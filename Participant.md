## 3. 参会人 (Participant)

### 3.1 参会人接口 (LocalParticipant interface)

参会人接口 LocalParticipant 继承了 Participant 接口，定义了房间内参会人的所有行为和可用操作。

该接口在 [pkg/rtc/types/interfaces.go](https://github.com/livekit/livekit/blob/master/pkg/rtc/types/interfaces.go#L294) 中定义。

### 3.2 参会人实现类 (ParticipantImpl)

参会人实现类 ParticipantImpl 实现了 参会人接口 LocalParticipant。代码位于 [pkg/rtc/participant.go](https://github.com/livekit/livekit/blob/master/pkg/rtc/participant.go#L140)

该源码实现了参会人的所有行为。

### 3.3 参会人流传输管理器类 TransportManager

ParticipantImpl 类内嵌 TransportManager 类指针，通过 TransportManager 类管理参会人发布和订阅的媒体流。该类定义如下:

```go
type TransportManager struct {
    params TransportManagerParams

    lock sync.RWMutex

    publisher               *PCTransport
    subscriber              *PCTransport
    failureCount            int
    isTransportReconfigured bool
    lastFailure             time.Time
    lastSignalAt            time.Time
    signalSourceValid       atomic.Bool

    pendingOfferPublisher        *webrtc.SessionDescription
    pendingDataChannelsPublisher []*livekit.DataChannelInfo
    lastPublisherAnswer          atomic.Value
    lastPublisherOffer           atomic.Value
    iceConfig                    *livekit.ICEConfig

    mediaLossProxy       *MediaLossProxy
    udpLossUnstableCount uint32
    signalingRTT, udpRTT uint32

    onPublisherInitialConnected        func()
    onSubscriberInitialConnected       func()
    onPrimaryTransportInitialConnected func()
    onAnyTransportFailed               func()

    onICEConfigChanged func(iceConfig *livekit.ICEConfig)
}
```

媒体流的订阅和发布由两个 WebRTC PeerConnection 分别负责，就是类定义中的 publisher 和 subscriber 成员，这两个成员变量都是 PCTransport 类类型。

PCTransport 类源码位于 pkg/rtc/transport.go ，它封装了 PION 库的 WebRTC PeerConnection 和 DataChannel，实现 WebRTC 媒体流管理。

### 3.4 参会人上传(发布)媒体轨道管理器类 UpTrackManager

上传(发布)媒体轨道管理器类 UpTrackManager 类定义如下：

```go
// UpTrackManager manages all uptracks from a participant
type UpTrackManager struct {
    params UpTrackManagerParams

    closed bool

    // publishedTracks that participant is publishing
    publishedTracks               map[livekit.TrackID]types.MediaTrack
    subscriptionPermission        *livekit.SubscriptionPermission
    subscriptionPermissionVersion utils.TimedVersion
    // subscriber permission for published tracks
    subscriberPermissions map[livekit.ParticipantIdentity]*livekit.TrackPermission // subscriberIdentity => *livekit.TrackPermission

    lock sync.RWMutex

    // callbacks & handlers
    onClose        func()
    onTrackUpdated func(track types.MediaTrack)
}
```

### 3.5 参会人媒体轨道订阅管理类 SubscriptionManager

媒体轨道订阅管理类 SubscriptionManager 管理参会人订阅他人媒体流和自身媒体流被他人订阅的状态。该类定义如下：

```go
// SubscriptionManager manages a participant's subscriptions
type SubscriptionManager struct {
    params              SubscriptionManagerParams
    lock                sync.RWMutex
    subscriptions       map[livekit.TrackID]*trackSubscription // 轨道是订阅的情况
    pendingUnsubscribes atomic.Int32

    subscribedVideoCount, subscribedAudioCount atomic.Int32

    subscribedTo map[livekit.ParticipantID]map[livekit.TrackID]struct{} // 订阅的他人轨道
    reconcileCh  chan livekit.TrackID
    closeCh      chan struct{}
    doneCh       chan struct{}

    onSubscribeStatusChanged func(publisherID livekit.ParticipantID, subscribed bool)
}
```

该类管理参会人发布和订阅的所有媒体轨道的订阅情况，跟踪参会人发布的媒体轨道的变化和关闭，具体功能包括：

- 参会人发布的媒体轨道被其他参会人订阅的情况；

- 参会人订阅的房间中其他参会人的媒体轨道的情况；

- 完成媒体流的订阅；

- 提供回调函数通知订阅状态发生改变

### 3.6 参会人网络负载情况管理类 ParticipantTrafficLoad

网络负载情况管理类 ParticipantTrafficLoad 管理参会人的网络通信负载情况，包括 RTP 媒体轨道和 DataChannel 数据通道的网络负载情况。类定义如下：

```go
type ParticipantTrafficLoadParams struct {
    Participant      *ParticipantImpl
    DataChannelStats *telemetry.BytesTrackStats
    Logger           logger.Logger
}

type ParticipantTrafficLoad struct {
    params ParticipantTrafficLoadParams

    lock               sync.RWMutex
    onTrafficLoad      func(trafficLoad *types.TrafficLoad)
    tracksStatsMedia   map[livekit.TrackID]*livekit.RTPStats
    dataChannelTraffic *telemetry.TrafficTotals
    trafficLoad        *types.TrafficLoad

    closed core.Fuse
}
```


