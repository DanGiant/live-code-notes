# livekit-server 的 Room 管理逻辑

LiveKit Server是一个 WebRTC SFU (Selective Forward Unit) 流媒体服务器。它以房间 (Room) 为最小单位组织音视频通话，即在同一个音视频多讲或音视频会议中的参与者 (Participant) 都在同一房间，由 Room 管理和控制 Participant 之间的音视频互通和控制。

## [1. LiveKit 的房间管理器 (RoomManager)](./RoomManager.md)

## [2. 房间 (即会议室，Room)](./Room.md)

## [3. 参会人 (Participant)](./Participant.md)

## [4. 参会人 (Participant) 加入房间](./Participant_Join_Room.md)

## [5. 参会人发布媒体](./Participant_Publish_MediaTracks.md)

## [6. 参会人订阅媒体](./Participant_Subscribe_MediaTracks.md)

## [7. 参会人之间进行数据通信](./Participant_Data_Communication.md)

## [8. 关闭房间](./Close_the_Room.md)
