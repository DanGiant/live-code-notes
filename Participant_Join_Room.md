## 3. 参会者 (Participant) 加入房间

### 3.1 StartSession() 的调用路径

RoomManager.StartSession() 函数的作用是启动参会者的会议会话。每个参会者 (Participant) 都通过调用 RoomManager.StartSession() 函数进入房间参加会议。StartSession() 函数为新加入的参会者运行一个单独的 rtcSessionWorker go 协程，处理该参会者的消息和事件。

函数 StartSession() 的源码位于 /pkg/service/roommanager.go 中，定义如下：

```go
// StartSession starts WebRTC session when a new participant is connected, takes place on RTC node
func (r *RoomManager) StartSession(
    ctx context.Context,
    roomName livekit.RoomName,
    pi routing.ParticipantInit,
    requestSource routing.MessageSource,
    responseSink routing.MessageSink,
) error
```

livekit-server 支持多个入口、多种方式创建会议，比如通过 livekit-client SDK 让后台程序以客户端方式通过 HTTP 协议访问 Room API，以及前端各平台的客户端调用 LiveKit SDK API 通过 websocket 协议连接 livekit-server 服务器。根据接入平台、接入方式的不同，有不同的 StartSession() 函数调用路径，如下图所示：

```mermaid
graph TB
    ClientSDK.connect--websocket-->RTCService.ServeHTTP-->RTCService.startConnection
    RTCService.startConnection-->router.StartParticipantSignal

    Call_Room_API--http-->RoomService.CreateRoom-->router.StartParticipantSignal

    router.StartParticipantSignal-->LocalRouter.StartParticipantSignal
    router.StartParticipantSignal-->RedisRouter.StartParticipantSignal

    LocalRouter.StartParticipantSignal-->LocalRouter.StartParticipantSignalWithNodeID
    RedisRouter.StartParticipantSignal-->LocalRouter.StartParticipantSignalWithNodeID

    LocalRouter.StartParticipantSignalWithNodeID-->SignalClient.StartParticipantSignal

    SignalClient.StartParticipantSignal--livekit.StartSession-->RTCNodeSink.WriteMessage

    RTCNodeSink.WriteMessage--livekit.StartSession-->redis.publishRTCMessage
    redis.publishRTCMessage--livekit.RTCNodeMessage_StartSession-->Redis.rtcChannel
    Redis.rtcChannel--livekit.RTCNodeMessage_StartSession-->RedisRouter.handleRTCMessage
    RedisRouter.handleRTCMessage--livekit.StartSession-->RedisRouter.startParticipantRTC
    RedisRouter.startParticipantRTC-->RedisRouter.onNewParticipant-->LocalRouter.OnNewParticipantRTC
    LocalRouter.OnNewParticipantRTC-->RoomManager.StartSession

    main-->service.InitializeServer
    service.InitializeServer-->service.NewDefaultSignalServer
    service.NewDefaultSignalServer-->RoomManager.StartSession
```

### 3.2 StartSession() 函数的主要流程

StartSession() 函数的主要流程如下图所示：

```mermaid
graph TB
    StartSession_called-->getOrCreateRoom-->room.GetParticipant
    
    room.GetParticipant--participant exist-->room.ResumeParticipant
    
    room.ResumeParticipant--send ReconnectResponse-->client
    
    room.ResumeParticipant--failed-->StartSession_returns_error
    room.ResumeParticipant--success-->start_r.rtcSessionWorker

    room.GetParticipant--participant not exist-->rtc.NewParticipant

    rtc.NewParticipant-->room.Join
    
    room.Join-->start_r.rtcSessionWorker-->StartSession_returns_success
    room.Join--send JoinResponse-->client
```

- 首先调用 getOrCreateRoom() 函数获取参会者要加入的房间对象，然后在房间中查找该参会人是否存在（如果参会人是掉线后重新进入就可能存在这种情况）。

- 如果参会人不存在，就生成参会人对象并调用 room.Join() 函数将参会人加入到房间，同时向客户端返回 JoinResponse 应答。

- 如果参会人存在且能够恢复（参会人短时间内掉线并重新进入房间），就向客户端返回 ReconnectResponse 应答；否则向客户端报告错误并返回。

- 如果参会人加入房间成功，StartSession() 函数为参会人启动 rtcSessionWorker go 协程并返回。

### 3.3 getOrCreateRoom() 函数：获取房间，或创建新房间

StartSession() 函数首先将为刚加入的参会者，根据其入会请求的 Token 中携带的欲加入的房间的 RoomName 调用 getOrCreateRoom() 函数，查找房间对象。如果房间还未创建，getOrCreateRoom 将会创建名为 RoomName 的房间并返回房间对象。
</br>

getOrCreateRoom() 函数的定义如下：

```go
func (r *RoomManager) getOrCreateRoom(ctx context.Context, roomName livekit.RoomName) (*rtc.Room, error)
```

#### 3.3.1 获取存在的房间

该函数先根据房间名RoomName获取房间对象。如果房间存在则直接返回。

```go
    r.lock.RLock()
    lastSeenRoom := r.rooms[roomName]
    r.lock.RUnlock()

    if lastSeenRoom != nil && lastSeenRoom.Hold() {
        return lastSeenRoom, nil
    }
```

#### 3.3.2 房间不存在，为创建房间获取房间信息

如房间不存在，就先调用 ServiceStore 接口的 LoadRoom() 函数获取房间信息。RoomManager 的成员 roomStore 实现了 ObjectStore 接口（ObjectStore接口继承 ServiceStore 接口）的实例，会根据实例的不同，调用不同版本的 LoadRoom() 实现。由于 livekit-server 支持本地存储和 Redis 存储两种方式启动，房间信息也根据启动 livekit-server 时配置文件 config.yaml 中 redis 存储的启用与否，调用不同版本（LocalStore 或者 RedisStore）的 ObjectStore 接口实现。

```go
    // create new room, get details first
    ri, internal, err := r.roomStore.LoadRoom(ctx, roomName, true)
    if err != nil {
        return nil, err
    }
```

#### 3.2.3 根据房间信息创建新房间


```go
    r.lock.Lock()

    currentRoom := r.rooms[roomName]
    for currentRoom != lastSeenRoom {
        r.lock.Unlock()
        if currentRoom != nil && currentRoom.Hold() {
            return currentRoom, nil
        }

        lastSeenRoom = currentRoom
        r.lock.Lock()
        currentRoom = r.rooms[roomName]
    }

    // construct ice servers
    newRoom := rtc.NewRoom(ri, internal, *r.rtcConfig, &r.config.Audio,
                           r.serverInfo, r.telemetry, r.agentClient, r.egressLauncher)
```

#### 3.3.4 注册房间关闭回调函数 Room.OnClose()

```go
    roomTopic := rpc.FormatRoomTopic(roomName)
    roomServer := utils.Must(rpc.NewTypedRoomServer(r, r.bus))
    killRoomServer := r.roomServers.Replace(roomTopic, roomServer)
    if err := roomServer.RegisterAllRoomTopics(roomTopic); err != nil {
        killRoomServer()
        r.lock.Unlock()
        return nil, err
    }

    newRoom.OnClose(func() {
        killRoomServer()

        roomInfo := newRoom.ToProto()
        r.telemetry.RoomEnded(ctx, roomInfo)
        prometheus.RoomEnded(time.Unix(roomInfo.CreationTime, 0))
        if err := r.deleteRoom(ctx, roomName); err != nil {
            newRoom.Logger.Errorw("could not delete room", err)
        }

        newRoom.Logger.Infow("room closed")
    })
```

#### 3.3.5 注册房间更新回调函数

```go
    newRoom.OnRoomUpdated(func() {
        if err := r.roomStore.StoreRoom(ctx, newRoom.ToProto(), newRoom.Internal()); err != nil {
            newRoom.Logger.Errorw("could not handle metadata update", err)
        }
    })
```

#### 3.3.6 注册参会者变化通知回调函数

```go
    newRoom.OnParticipantChanged(func(p types.LocalParticipant) {
        if !p.IsDisconnected() {
            if err := r.roomStore.StoreParticipant(ctx, roomName, p.ToProto()); err != nil {
                newRoom.Logger.Errorw("could not handle participant change", err)
            }
        }
    })
```

#### 3.3.7 更新RoomManager的房间列表，增加房间的引用计数

```go
    r.rooms[roomName] = newRoom

    r.lock.Unlock()

    newRoom.Hold()
```

### 3.4 创建参会人对象

#### 3.4.1 只创建房间不创建参会人

如果调用 StartSession() 函数时传入的参会人标识（ParticipantInfo 的 Identity 成员）为空，StartSession 将只创建房间，不会创建参会人对象，自然也没有参会人的 rtcWorkerSession 协程。

#### 3.4.2 获取存在的参会人对象

如果调用 StartSession() 函数时传入了参会人标识 (ParticipantIdentity)，将根据参会者的标识从房间查找参会人对象。

```go
    participant := room.GetParticipant(pi.Identity)
```

如果是已经加入房间的参会人，则会根据调用 StartSession() 时传入的 routing.ParticipantInit 指定的参会人是否因为断开重连的原因（pi.Reconnect）启动本次会话。

#### 3.4.3 创建新的参会人对象

如果房间中不存在该参会人，则为参会人配置相关信息，调用 rtc.NewParticipant() 函数创建新参会人实例。

```go
    participant, err = rtc.NewParticipant(rtc.ParticipantParams{
        Identity:                pi.Identity,
        Name:                    pi.Name,
        SID:                     sid,
        Config:                  &rtcConf,
        Sink:                    responseSink,
        AudioConfig:             r.config.Audio,
        VideoConfig:             r.config.Video,
        ProtocolVersion:         pv,
        Telemetry:               r.telemetry,
        Trailer:                 room.Trailer(),
        PLIThrottleConfig:       r.config.RTC.PLIThrottle,
        CongestionControlConfig: r.config.RTC.CongestionControl,
        PublishEnabledCodecs:    protoRoom.EnabledCodecs,
        SubscribeEnabledCodecs:  protoRoom.EnabledCodecs,
        Grants:                  pi.Grants,
        Logger:                  pLogger,
        ClientConf:              clientConf,
        ClientInfo:              rtc.ClientInfo{ClientInfo: pi.Client},
        Region:                  pi.Region,
        AdaptiveStream:          pi.AdaptiveStream,
        AllowTCPFallback:        allowFallback,
        TURNSEnabled:            r.config.IsTURNSEnabled(),
        GetParticipantInfo: func(pID livekit.ParticipantID) *livekit.ParticipantInfo {
            if p := room.GetParticipantByID(pID); p != nil {
                return p.ToProto()
            }
            return nil
        },
        ReconnectOnPublicationError:  reconnectOnPublicationError,
        ReconnectOnSubscriptionError: reconnectOnSubscriptionError,
        ReconnectOnDataChannelError:  reconnectOnDataChannelError,
        DataChannelMaxBufferedAmount: r.config.RTC.DataChannelMaxBufferedAmount,
        VersionGenerator:             r.versionGenerator,
        TrackResolver:                room.ResolveMediaTrackForSubscriber,
        SubscriberAllowPause:         subscriberAllowPause,
        SubscriptionLimitAudio:       r.config.Limit.SubscriptionLimitAudio,
        SubscriptionLimitVideo:       r.config.Limit.SubscriptionLimitVideo,
        PlayoutDelay:                 roomInternal.GetPlayoutDelay(),
        SyncStreams:                  roomInternal.GetSyncStreams(),
    })
    if err != nil {
        return err
    }
```

#### 3.4.4 参会人加入房间

创建新参会人实例之后，StartSession() 函数调用下面的代码将新参会人加入房间。

```go
    // join room
    opts := rtc.ParticipantOptions{
        AutoSubscribe: pi.AutoSubscribe,
    }
    iceServers := r.iceServersForParticipant(apiKey, participant,
        iceConfig.PreferenceSubscriber == livekit.ICECandidateType_ICT_TLS)

    if err = room.Join(participant, requestSource, &opts, iceServers); err != nil {
        pLogger.Errorw("could not join room", err)
        _ = participant.Close(true, types.ParticipantCloseReasonJoinFailed, false)
        return err
    }
```

参会人加入房间时，会同时设置是否自动订阅 (AutoSubscribe)，并设置参会人的 ICE 服务器。
加入房间通过调用 room.Join() 函数完成，该函数请见第 [3.5 将参会人加入房间](#35-roomjoin-将参会人加入房间) 节。

#### 3.4.5 设置新参会者的OnClose回调函数

```go
    participant.OnClose(func(p types.LocalParticipant) {
        killParticipantServer()

        if err := r.roomStore.DeleteParticipant(ctx, roomName, p.Identity()); err != nil {
            pLogger.Errorw("could not delete participant", err)
        }

        // update room store with new numParticipants
        proto := room.ToProto()
        persistRoomForParticipantCount(proto)
        r.telemetry.ParticipantLeft(ctx, proto, p.ToProto(), true)
    })
```

#### 3.4.6 设置新参会者的其他回调函数

```go
    participant.OnClaimsChanged(func(participant types.LocalParticipant) {
        pLogger.Debugw("refreshing client token after claims change")
        if err := r.refreshToken(participant); err != nil {
            logger.Errorw("could not refresh token", err)
        }
    })
    participant.OnICEConfigChanged(func(participant types.LocalParticipant, iceConfig *livekit.ICEConfig) {
        r.lock.Lock()
        r.iceConfigCache[participant.Identity()] = &iceConfigCacheEntry{
            iceConfig:  iceConfig,
            modifiedAt: time.Now(),
        }
        r.lock.Unlock()
    })
```

### 3.5 room.Join() 将参会人加入房间

room.Join() 函数完成了下面的工作:

```mermaid
graph TB
    room.Join_bigins-->check_Room.IsClosed-->check_Parcipant_AlreadJoined-->check_Participant_MaxNumber
    check_Participant_MaxNumber-->set_Participant_callbacks-->save_Participant_to_Room-->send_JoinResponse_to_Client-->exec_Subscribe_for_Participant-->room.Join_ends
```

- 先检查能否将参会人加入房间
- 如果可以，将为参会人设置各个与媒体发布、订阅还有数据相关的回调函数，包括：

```go
    // it's important to set this before connection, we don't want to miss out on any published tracks
    participant.OnTrackPublished(r.onTrackPublished)
    participant.OnStateChange(func(p types.LocalParticipant, oldState livekit.ParticipantInfo_State) {
        ...
    })
    participant.OnTrackUpdated(r.onTrackUpdated)
    participant.OnTrackUnpublished(r.onTrackUnpublished)
    participant.OnParticipantUpdate(r.onParticipantUpdate)
    participant.OnDataPacket(r.onDataPacket)
    participant.OnSubscribeStatusChanged(func(publisherID livekit.ParticipantID, subscribed bool) {
        ...
    })
```

- 然后将参会人保存到房间中，主要是写入 Room 实例的各 participants map[]；

```go
    r.participants[participant.Identity()] = participant
    r.participantOpts[participant.Identity()] = opts
    r.participantRequestSources[participant.Identity()] = requestSource
```

- 调用 Room.onParticipantChanged() 回调函数，向房间内的所有其他参会人通知有新参会人进入房间；
- 向客户端发送 JoinResponse 应答。该应答是新加入参会人的 ws 连接收到的第一个应答；
- 在参会人设置了 SubscriberAsPrimary (订阅作为主 PeerConnection) 时，为参会人启动订阅，订阅房间内的其他参会人的媒体轨。

### 3.6 为新参会者启动go协程

```go
go r.rtcSessionWorker(room, participant, requestSource)
```

下面是协程 rtcSessionWorker 的主要功能逻辑：

```go
// manages an RTC session for a participant, runs on the RTC node
func (r *RoomManager) rtcSessionWorker(room *rtc.Room, participant types.LocalParticipant, requestSource routing.MessageSource) {

    ...

    // send first refresh for cases when client token is close to expiring
    _ = r.refreshToken(participant)
    tokenTicker := time.NewTicker(tokenRefreshInterval)
    defer tokenTicker.Stop()
    stateCheckTicker := time.NewTicker(time.Millisecond * 500)
    defer stateCheckTicker.Stop()
    for {
        select {
        case <-stateCheckTicker.C:
            // periodic check to ensure participant didn't become disconnected
            if participant.IsDisconnected() {
                return
            }
        case <-tokenTicker.C:
            // refresh token with the first API Key/secret pair
            if err := r.refreshToken(participant); err != nil {
                pLogger.Errorw("could not refresh token", err, "connID", requestSource.ConnectionID())
            }
        case obj := <-requestSource.ReadChan():
            
            ...

            req := obj.(*livekit.SignalRequest)
            if err := rtc.HandleParticipantSignal(room, participant, req, pLogger); err != nil {
                // more specific errors are already logged
                // treat errors returned as fatal
                return
            }
        }
    }
}
```

协程 RoomManager.rtcSessionWorker() 负责的工作是：

- 每间隔 tokenRefreshInterval (默认设置是5分钟) 更新参会者的 token;
- 每间隔 500ms 检查参会者是否在线;
- 从channel读取参会者的请求，调用 HandleParticipantSignal() 函数进行相应的处理。
