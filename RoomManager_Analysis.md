# livekit-server 的 Room 管理逻辑

LiveKit Server是一个 WebRTC SFU (Selective Forward Unit) 流媒体服务器。它以房间 (Room) 为最小单位组织音视频通话，即在同一个音视频多讲或音视频会议中的参与者 (Participant) 都在同一房间，由 Room 管理和控制 Participant 之间的音视频互通和控制，livekit-server 通过 RoomManager 管理服务器上的所有 Room。

## 1. 房间管理器 (RoomManager) 类的定义

房间管理器 (RoomManager) 类在源文件 pkg/service/roommanager.go 中定义，代码如下所示：

```go
// RoomManager manages rooms and its interaction with participants.
// It's responsible for creating, deleting rooms, as well as running sessions for participants
type RoomManager struct {
    lock sync.RWMutex

    config            *config.Config
    rtcConfig         *rtc.WebRTCConfig
    serverInfo        *livekit.ServerInfo
    currentNode       routing.LocalNode
    router            routing.Router
    roomStore         ObjectStore
    telemetry         telemetry.TelemetryService
    clientConfManager clientconfiguration.ClientConfigurationManager
    agentClient       rtc.AgentClient
    egressLauncher    rtc.EgressLauncher
    versionGenerator  utils.TimedVersionGenerator
    turnAuthHandler   *TURNAuthHandler
    bus               psrpc.MessageBus

    rooms map[livekit.RoomName]*rtc.Room

    roomServers        utils.MultitonService[rpc.RoomTopic]
    participantServers utils.MultitonService[rpc.ParticipantTopic]

    iceConfigCache map[livekit.ParticipantIdentity]*iceConfigCacheEntry
}
```

RoomManager 用字典 map[] 管理服务器中的房间，以房间名 livekit.RoomName 作为 key 值检索管理的 Room。

```go
    rooms map[livekit.RoomName]*rtc.Room
```

## 2. 创建 RoomManager

RoomManager 在livekit-server 启动的时候创建，其调用流程如下：

```mermaid
graph LR
    main-->startServer-->service.InitializeServer-->NewLocalRoomManager
```

## 3. 参会者 (Participant) 加入房间

### 3.1 调用 StartSession() 启动参会者Session

每个参会者 (Participant) 加入会议都需要调用 RoomManager.StartSession() 函数，为该 Participant 创建一个单独的 Session 并启动一个go协程单独处理该 Participant 的消息。函数 StartSession() 的源码位于 /pkg/service/roommanager.go 中，定义如下：

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

由于 livekit-server 支持多个入口、多种方式创建会议，比如在后台通过 livekit-client SDK 以客户端方式通过 HTTP 协议访问 Room API，以及前端通过各平台的 LiveKit 客户端 SDK 通过 websocket 协议连接 livekit-server 服务器。根据接入平台、接入方式的不同，有不同的 StartSession() 函数调用路径，如下图所示：

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

### 3.2 获取或创建房间 (getOrCreateRoom)

在 StartSession() 函数首先将为加入的参会者根据其欲加入的房间的 RoomName 调用 getOrCreateRoom() 函数获取房间对象。如果房间还未创建，将先创建名字为 RoomName 的房间。getOrCreateRoom() 函数的定义如下：

```go
func (r *RoomManager) getOrCreateRoom(ctx context.Context, roomName livekit.RoomName) (*rtc.Room, error)
```

#### 3.2.1 获取存在的房间

该函数先根据房间名RoomName获取房间对象。如果房间存在则直接返回。

```go
    r.lock.RLock()
    lastSeenRoom := r.rooms[roomName]
    r.lock.RUnlock()

    if lastSeenRoom != nil && lastSeenRoom.Hold() {
        return lastSeenRoom, nil
    }
```

#### 3.2.2 房间不存在，为创建房间获取房间信息

如房间不存在，就先调用 ServiceStore 接口的 LoadRoom() 函数获取房间信息。RoomManager的成员roomStore是实现了 ObjectStore 接口（ObjectStore接口继承了ServiceStore接口）的实例，会根据实例的不同，调用不同版本的 LoadRoom() 实现。由于 livekit-server 支持本地存储和 Redis 存储两种方式启动，房间信息也根据启动 livekit-server 时配置文件 config.yaml 中 redis 存储的启用与否，会调用不同版本（LocalStore或者RedisStore）的 ObjectStore 接口实现。

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

#### 3.2.4 注册房间关闭回调函数 Room.OnClose()

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

#### 3.2.5 注册房间更新回调函数

```go
    newRoom.OnRoomUpdated(func() {
        if err := r.roomStore.StoreRoom(ctx, newRoom.ToProto(), newRoom.Internal()); err != nil {
            newRoom.Logger.Errorw("could not handle metadata update", err)
        }
    })
```

#### 3.2.6 注册参会者变化通知回调函数

```go
    newRoom.OnParticipantChanged(func(p types.LocalParticipant) {
        if !p.IsDisconnected() {
            if err := r.roomStore.StoreParticipant(ctx, roomName, p.ToProto()); err != nil {
                newRoom.Logger.Errorw("could not handle participant change", err)
            }
        }
    })
```

#### 3.2.7 更新RoomManager的房间列表，增加房间的引用计数

```go
    r.rooms[roomName] = newRoom

    r.lock.Unlock()

    newRoom.Hold()
```


### 3.3 创建参会者 Session 并为其启动 go 协程

#### 3.3.1 只创建房间不启动参会者会话

如果调用 StartSession() 函数时传入的参会人标识（ParticipantInfo 的 Identity 成员）为空，StartSession 将只创建房间，不会启动 Participant Session。

#### 3.3.2 获取存在的参会者对象

否则，将根据参会者的 Identity 从房间获取参会者对象。

```go
    participant := room.GetParticipant(pi.Identity)
```

如果是房间中存在的参会者，则会根据调用 StartSession() 时传入的 routing.ParticipantInit 指定的参会者是否因为断开重连原因（pi.Reconnect）启动本次会话，决定是进行会话重连还是抛弃前一次会话。

#### 3.3.3 参会者不存在，创建新参会者

如果是新加入的参会者，则为参会者配置相关信息，调用 rtc.NewParticipant() 函数创建新参会者。

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

#### 3.3.4 设置新参会者的OnClose回调函数

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

#### 3.3.5 设置新参会者的其他回调函数

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

#### 3.3.5 为新参会者启动go协程

```go
go r.rtcSessionWorker(room, participant, requestSource)
```

协程 RoomManager.rtcSessionWorker() 负责的工作是：

1. 每间隔 tokenRefreshInterval (默认设置是5分钟) 更新参会者的 token;
2. 每间隔 500ms 检查参会者是否在线;
3. 从channel读取参会者的请求，调用 HandleParticipantSignal() 函数进行相应的处理。

## 4. 关闭房间

### 4.1 Room.Close() 函数

Room.Close() 函数负责关闭房间。它首先检查房间是否已经关闭，如果已经关闭则直接返回不进行任何操作。如果房间尚未关闭，将首先为房间内的每个参会者调用Close()，然后调用房间的onClose()回调函数，完成房间的关闭逻辑。

```go
func (r *Room) Close() {
    r.lock.Lock()
    select {
    case <-r.closed:
        r.lock.Unlock()
        return
    default:
        // fall through
    }
    close(r.closed)
    r.lock.Unlock()
    r.Logger.Infow("closing room")
    for _, p := range r.GetParticipants() {
        _ = p.Close(true, types.ParticipantCloseReasonRoomClose, false)
    }
    r.protoProxy.Stop()
    if r.onClose != nil {
        r.onClose()
    }
}
```

### 4.2 多个调用 Room.Close() 函数的路径

Room.Close() 函数是最终完成关闭房间操作的逻辑，在 livekit-server 的代码中有多个地方会调用到它。

下图展示了调用该函数的多个入口的调用路径：


```mermaid
graph TB
    HttpClient--request DeleteRoom-->roomServiceServer.ServeHTTP
    roomServiceServer.ServeHTTP--DeleteRoom-->roomServiceServer.serveDeleteRoom
    roomServiceServer.serveDeleteRoom-->RoomService.DeleteRoom
    RoomService.DeleteRoom--livekit.RTCNodeMessage_DeleteRoom-->LocalRouter.rtcMessageWorker
    RoomService.DeleteRoom--livekit.RTCNodeMessage_DeleteRoom-->RedisRouter.redisWorker

    LocalRouter.rtcMessageWorker-->LocalRouter.OnRTCMessage
    RedisRouter.redisWorker-->RedisRouter.handleRTCMessage
    RedisRouter.handleRTCMessage-->LocalRouter.OnRTCMessage
    LocalRouter.OnRTCMessage--livekit.RTCNodeMessage_DeleteRoom-->RoomManager.handleRTCMessage
    RoomManager.handleRTCMessage-->RoomManager.DeleteRoom
    RoomManager.DeleteRoom-->Room.Close

    LivekitServer.main-->startServer-->LivekitServer.Start
    LivekitServer.Start--Shutdowned-->RoomManager.Stop
    RoomManager.Stop-->Room.Close

    LivekitServer.Start--start go routine-->LivekitServer.backgroundWorker
    LivekitServer.backgroundWorker-->RoomManager.CloseIdleRooms
    RoomManager.CloseIdleRooms-->Room.CloseIfEmpty
    Room.CloseIfEmpty-->Room.Close
```
