## 2. 房间 (即会议室，Room)

### 2.1 房间 (Room) 的类定义

房间 (Room) 类在 pkg/rtc/room.go 文件中定义，代码如下：

```go
type Room struct {
    lock sync.RWMutex

    protoRoom  *livekit.Room
    internal   *livekit.RoomInternal
    protoProxy *utils.ProtoProxy[*livekit.Room]
    Logger     logger.Logger

    config         WebRTCConfig
    audioConfig    *config.AudioConfig
    serverInfo     *livekit.ServerInfo
    telemetry      telemetry.TelemetryService
    egressLauncher EgressLauncher
    trackManager   *RoomTrackManager

    // agents
    agentClient            AgentClient
    publisherAgentsEnabled bool

    // map of identity -> Participant
    participants              map[livekit.ParticipantIdentity]types.LocalParticipant
    participantOpts           map[livekit.ParticipantIdentity]*ParticipantOptions
    participantRequestSources map[livekit.ParticipantIdentity]routing.MessageSource
    hasPublished              map[livekit.ParticipantIdentity]bool
    bufferFactory             *buffer.FactoryOfBufferFactory

    // batch update participant info for non-publishers
    batchedUpdates   map[livekit.ParticipantIdentity]*livekit.ParticipantInfo
    batchedUpdatesMu sync.Mutex

    // time the first participant joined the room
    joinedAt atomic.Int64
    holds    atomic.Int32
    // time that the last participant left the room
    leftAt atomic.Int64
    closed chan struct{}

    trailer []byte

    onParticipantChanged func(p types.LocalParticipant)
    onRoomUpdated        func()
    onClose              func()

    // add by chenjian to support room parallel expansion
    onParallelRoomOnline  func()
    onParallelRoomOffline func()
}
```

### 2.2 房间 (Room) 类几个主要成员

#### 2.2.1 房间基础数据 livekit.Room

```go
    protoRoom  *livekit.Room
```

成员 protoRoom 是 livekit.Room 类型，该类型是用 protobuf 定义的 Room 数据对象，存储传递给客户端的房间信息。它在 livekit 的 protocol 项目中定义，源码是 livekit_models.proto。其 protobuf 定义如下：

```protobuf
message Room {
    string sid = 1;
    string name = 2;
    uint32 empty_timeout = 3;
    uint32 departure_timeout = 14;
    uint32 max_participants = 4;
    int64 creation_time = 5;
    string turn_password = 6;
    repeated Codec enabled_codecs = 7;
    string metadata = 8;
    uint32 num_participants = 9;
    uint32 num_publishers = 11;
    bool active_recording = 10;
    TimedVersion version = 13;

    // NEXT-ID: 15
}
```

protoc-gen-go 为 livekit_modles.proto 生成的 livekit_modles.pb.go 代码里，message Room 的 go 实现是 type Room struct:

```go
package livekit

...

type Room struct {
    ...

    Sid             string   `protobuf:"bytes,1,opt,name=sid,proto3" json:"sid,omitempty"`
    Name            string   `protobuf:"bytes,2,opt,name=name,proto3" json:"name,omitempty"`
    EmptyTimeout    uint32   `protobuf:"varint,3,opt,name=empty_timeout,json=emptyTimeout,proto3" json:"empty_timeout,omitempty"`
    MaxParticipants uint32   `protobuf:"varint,4,opt,name=max_participants,json=maxParticipants,proto3" json:"max_participants,omitempty"`
    CreationTime    int64    `protobuf:"varint,5,opt,name=creation_time,json=creationTime,proto3" json:"creation_time,omitempty"`
    TurnPassword    string   `protobuf:"bytes,6,opt,name=turn_password,json=turnPassword,proto3" json:"turn_password,omitempty"`
    EnabledCodecs   []*Codec `protobuf:"bytes,7,rep,name=enabled_codecs,json=enabledCodecs,proto3" json:"enabled_codecs,omitempty"`
    Metadata        string   `protobuf:"bytes,8,opt,name=metadata,proto3" json:"metadata,omitempty"`
    NumParticipants uint32   `protobuf:"varint,9,opt,name=num_participants,json=numParticipants,proto3" json:"num_participants,omitempty"`
    NumPublishers   uint32   `protobuf:"varint,11,opt,name=num_publishers,json=numPublishers,proto3" json:"num_publishers,omitempty"`
    ActiveRecording bool     `protobuf:"varint,10,opt,name=active_recording,json=activeRecording,proto3" json:"active_recording,omitempty"`
}
```

livekit.Room类各成员的作用如下：

```txt
Sid:             服务器给房间分配的ID，即ServerID。
Name:            房间名。
EmptyTimeout:    检测到房间中无人时关闭房间的延后时间。
MaxParticipants: 房间支持的最大参会者数。
TurnPassword:    TURN中继服务器的密码。如果启用了TURN服务器并设置了访问密码，需要向客户端传输该数据。
EnabledCodecs:   服务器支持的音频和视频的编码格式列表。
Metadata:        房间的元数据。
NumParticipants: 当家房间中的参会者数。
ActiveRecording: 是否正在进行录像。
```

#### 2.2.2 服务器信息 ServerInfo

参会者接入房间收到的第一个回复 JoinResponse 中包含该信息，用于向客户端告知当前服务器的类型、版本、节点ID等信息。

服务器信息的 protobuf 定义：

```protobuf
// details about the server
message ServerInfo {
  enum Edition {
    Standard = 0;
    Cloud = 1;
  }
  Edition edition = 1;
  string version = 2;
  int32 protocol = 3;
  string region = 4;
  string node_id = 5;
  // additional debugging information. sent only if server is in development mode
  string debug_info = 6;
}
```

ServerInfo 中需要关注的重要信息有：

```txt
Edition: livekit-server 服务器的版本类型。Standard是标准的开源版，Cloud是云服务器版；

Version: livekit-server 服务器的版本号。

NodeId:  livekit-server 的节点ID。
```

#### 2.2.3 房间媒体流轨道管理器 RoomTrackManager

```go
    trackManager   *RoomTrackManager
```

RoomTrackManager 类在 pkg/rtc/roommanager.go 中定义：

```go
// RoomTrackManager holds tracks that are published to the room
type RoomTrackManager struct {
    lock            sync.RWMutex
    changedNotifier *utils.ChangeNotifierManager
    removedNotifier *utils.ChangeNotifierManager
    tracks          map[livekit.TrackID]*TrackInfo
}
```

房间媒体流轨道管理器负责房间内的所有已经发布媒体流（包括音频流和视频流）轨道。

```txt
tracks:  map[livekit.TrackID]*TrackInfo 类型，存储以 TrackID 为索引的媒体轨数据；

changedNotifier: 媒体轨的变化通知器，ChangeNotifierManager 类实例，存储 notifiers map[string]*ChangeNotifier 映射，管理所有已发布媒体流的变化通知器。参会人对媒体流的变更通过各自的媒体流 ChangeNotifier 通知到房间并由房间以广播方式转发。

removedNotifier: 媒体轨的删除通知器，ChangeNotifierManager 类实例，存储 notifiers map[string]*ChangeNotifier 映射，管理所有已发布媒体流的删除通知器。参会人对删除媒体流的事件通过各自的媒体流 ChangeNotifier 通知到房间并由房间以广播方式转发。
```

#### 2.2.4 管理参会者的成员变量

```go
    // map of identity -> Participant
    participants              map[livekit.ParticipantIdentity]types.LocalParticipant
    participantOpts           map[livekit.ParticipantIdentity]*ParticipantOptions
    participantRequestSources map[livekit.ParticipantIdentity]routing.MessageSource
    hasPublished              map[livekit.ParticipantIdentity]bool
```

上述4个成员变量管理房间中的参会者 (Participant)。均为 map[] 类型，以 ParticipantIdentity (其实是个重定义的 string 类型，存储参会者的唯一标识) 为索引。其作用如下：

```txt
participants: 存储房间中的参会人实例。声明中指定存储实现了 types.LocalParticipant 接口的类实例，实际存储的是 ParticipantImpl 类对象，实现代码位于 pkg/rtc/participant.go。

participantOpts: 存储参会人实例的配置，目前 ParticipantOptions 成员只有 AutoSubscribe (bool 型)，控制参会人是否自动订阅房间中发布的媒体流。

participantRequestSources: 存储参会人实例的 MessageSource 接口对象。用于监视参会人的业务请求、管理参会人的信令连接。

hasPublished: 存储参会人实例的媒体流发布状态。
```

#### 2.2.5 管理房间状态和生命周期的成员变量

```go
    // time the first participant joined the room
    joinedAt atomic.Int64
    holds    atomic.Int32
    // time that the last participant left the room
    leftAt atomic.Int64
    closed chan struct{}
```

```txt
joinedAt: 第一个参会者加入会议室的时间。

holds: 房间的引用计数器，用于控制房间关闭的变量。

leftAt: 最后一个参会者离开房间的时间。

closed: 用于接收房间是否关闭的 channel。

```

#### 2.2.6 与加密相关的成员变量

```go
    trailer []byte
```

在启用了 livekit-server 的 E2EE (end-to-end encryption，端到端加密) 加密的情况下，服务器在音视频帧数据的尾端添加的标志性数据，用于标识未加密帧。

#### 2.2.7 房间的回调函数

```go
    onParticipantChanged func(p types.LocalParticipant)
    onRoomUpdated        func()
    onClose              func()
```

```txt
onParticipantChanged: 通知有参会人发生变化。

onRoomUpdated: 通知房间信息发生了变化。

onClose: 通知房间已经关闭。
```

### 2.2 房间的
