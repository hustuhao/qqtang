# QQ堂多人对局同步机制研究（PVP / PVE）

> 研究对象：本仓库的 Go 兼容服务端及其原版客户端协议；不是腾讯原始服务端源码。  
> 研究日期：2026-10-25。结论按“代码直接证实 / 合理推断 / 尚待验证”分类。

## 1. 初步结论

**代码直接证实**：服务端采用 TCP 登录/业务通道、QQTPPP UDP 实时中继、TCP 房间快速包 0x0065 可靠上传的组合；正式对局 TCP 快速包可以是相同场景事件的镜像，不能误认为所有事件都应再次广播。纯真人房间的 UDP 场景包经身份/成员验证后原样转发，保留包序号、批次索引和顺序，服务端不逐包执行完整的真人移动/爆炸模拟。见 [协议说明](protocol-overview.md)、[room_peer_udp.go](../internal/server/probe/room_peer_udp.go) 的 `handleLegacyUDPMulticast`、[room_fast_dispatch.go](../internal/server/probe/room_fast_dispatch.go) 的 `handleRoomFastPacket`。

**代码直接证实**：对局层选择真人仲裁者（arbitrator），原生客户端执行部分共享场景规则；Go 服务端维护参与者、存活/死亡、目标进度与结算。普通竞技淘汰允许服务端根据队伍存活情况结束比赛，部分特殊地图的最终 `GAME_OVER` 来自客户端原生规则并由服务端验证。见 [competitive.go](../internal/game/match/competitive.go)、[adventure.go](../internal/game/match/adventure.go)。

**归类（研究判断）**：更接近“客户端场景演算 + 实时事件转发 + 选举式共享场景仲裁 + 服务端部分权威”，不应在缺少客户端动态证据时直接定性为纯帧同步或全服务端权威状态同步。也不能把当前重建工程的实现直接视为腾讯历史服务器的真实实现。

## 2. 网络路径与去重边界

1. TCP 登录绑定认证身份；QQTPPP UDP 登记/验证源端点。
2. `handleLegacyUDPMulticast` 使用 `resolveLegacyUDPMulticastTargets` 验证发送者和当前房间归属；纯真人接收方拿到原样密文。只有包明确面向服务端 AI 虚拟玩家时，才额外解码并输入 AI 战斗引擎。
3. TCP 0x0065 上传经 `handleRoomFastPacketLocked` 校验 UIN、房间阶段、GameID 和成员归属；它能转换为客户端可消费的 QQTPPP Type-2 数据包。正式对局事件不能因此在 UDP/TCP 两条路径上无条件重放。
4. `normalizeCompetitiveNativeRelayFastPackages` 对部分原生拾取、死亡通知执行同一事件去重；该行为不能外推为所有事件统一幂等。

相关文件：
- [protocol-overview.md](protocol-overview.md)
- [room_peer_udp.go](../internal/server/probe/room_peer_udp.go)
- [room_fast_dispatch.go](../internal/server/probe/room_fast_dispatch.go)
- [room_fast_packet.go](../internal/protocol/game/room_fast_packet.go)

## 3. 双人 PVP 案例：放泡 → 爆炸 → 死亡 → 结算

假设 A、B 位于同一普通淘汰竞技房间。以下是**逻辑事件顺序**，不是已验证的具体客户端线程调用栈；UDP 接收顺序和真实时间仍需抓包确认。

| 阶段 | 消息 / 编码证据 | 当前能证实的行为 | 未证实之处 |
| --- | --- | --- | --- |
| A 放泡 | `PLAYER_USE_BOMB = 0x0FA3`，15 字节正文 | `PlayerID:uint16, ClientTime:uint32, BombID:uint32, Row:uint8, Column:uint8, Power:uint16, Property:uint8`；网络大端 | 客户端何时生成、是否预测、谁检查放泡位置合法性 |
| 爆炸 | `NOTIFY_BOMB_EXPLODE = 0x0FA4`，变长正文 | 包含事件玩家、时间、爆炸泡列表、被破坏地图元素列表、被移除的场景道具列表 | 谁触发倒计时到期；仲裁者与所有客户端是否各自计算火焰 |
| B 死亡 | `0x0FA7` 玩家死亡事件 | `handleCompetitiveDeath` 解析死亡事件并更新 `CompetitiveBattle.RecordDeath`；对原生耐久类规则使用单独授权判断 | 客户端谁检测火焰与角色碰撞、谁首先产生死亡事件 |
| 比赛结束 | `GAME_OVER` | 普通淘汰在胜负首次确定时构建 GAME_OVER；死亡可靠镜像不能再产生第二次 TCP 死亡场景通知 | 实机广播顺序、动画延迟和丢包下重传行为 |

### 3.1 放泡：0x0FA3

源码：[bomb_event.go](../internal/protocol/game/bomb_event.go) 的 `PlayerUseBombEvent`、`ParsePlayerUseBombEvent`。

网络正文精确布局（全部多字节数值使用 BigEndian）：

```text
offset  size   field
0       2      PlayerID
2       4      ClientTime
6       4      BombID
10      1      Row
11      1      Column
12      2      Power
14      1      Property
total  15
```

说明：此处 `BombID` 是 32-bit 标识，不能与爆炸列表项中的 1-byte `BombID` 在没有证据时视为同一个字段。

### 3.2 爆炸：0x0FA4

源码：[bomb_event.go](../internal/protocol/game/bomb_event.go) 的 `BombExplodeEvent`、`ParseBombExplodeEvent`。

```text
PlayerID:u16 | ClientTime:u32 | BombCount:u8
  Bombs[BombCount]                 // 每项 13 字节：
    PlayerID:u16 | ClientTime:u32 | BombID:u8
    Row:u8 | Column:u8 | RowMin:u8 | RowMax:u8 | ColumnMin:u8 | ColumnMax:u8
MapElemCount:u8
  MapElems[MapElemCount]           // 每项 4 字节：MapElementID:u16 | Row:u8 | Column:u8
ItemCount:u8
  Items[ItemCount]                 // 每项 6 字节：ItemID:u32 | Row:u8 | Column:u8
```

当前解析器限制：BombCount 1..64、MapElemCount ≤32、ItemCount ≤32。Items 是**已经可见并被移除的场景道具**，并非墙破坏后新暴露的隐藏道具；隐藏物品由场景种子/初始内容规则产生。爆炸消息携带火焰传播边界（RowMin/RowMax/ColumnMin/ColumnMax）以及场景破坏结果，**证明同步数据不只是原始按键输入**。是否存在额外的本地确定性模拟，仍待验证。

### 3.3 死亡 → GAME_OVER

源码：[competitive_death_dispatch.go](../internal/server/probe/competitive_death_dispatch.go)。

`handleCompetitiveDeath` 对已经解析的 0x0FA7 事件：
1. 找出当前竞技战斗；检查死亡对象为玩家、已注册 Boss 或原生 NPC；
2. 普通参赛者调用 `battle.RecordDeath(death.PlayerID)`，特殊耐久模式先进行额外授权与生效确认；
3. 记录该事件已在 QQTPPP Type-2 多播中出现，TCP 可靠副本只做权威死亡记账，**不会额外广播相同死亡场景**；
4. 若 `resolution.NewlyConcluded`，生成 `GAME_OVER`，安排后续结算（代码中的普通原生结算展示延迟为 750ms；这不是网络 Tick 长度）。

源码：[match_game_over_dispatch.go](../internal/server/probe/match_game_over_dispatch.go)：对客户端原生 `GAME_OVER`，服务端进行比赛分类、来源与结果检查、必要的结果规范化及奖励结算。

### 3.4 相关的移动证据

[ movement_event.go ](../internal/protocol/game/movement_event.go) 中 `PLAYER_MOVE = 0x0FA2`，支持最多 16 条移动记录。每条记录包含 Sequence、TimeStamp、CurrentPos、CornerPos、EndPos、WalkAndDirection、Speed；其中方向/行走复合字节的位分配以及 SeqReply 的 ACK 语义尚未动态确认。这表明移动上传不等同于仅发送方向按键。

## 4. PVE 差异

[adventure.go](../internal/game/match/adventure.go) 的 `AdventureBattle` 按 GameID/MapID 保存参与者、存活状态和 `arbitratorID`。它明确要求 `REQUEST_GET_ITEM` 等共享场景请求只送至当前仲裁客户端。服务端 `RecordNPCDeath` 维护关卡死亡实例计数；观察到的 NPC 死亡消息不证明最后一击归属，因此不能凭空推断单个玩家击杀权威。关卡切换时存活者与已死亡需单独结算者有不同的通知/退出路径。

## 5. 证据分级和待完成任务

**已从源码证实**：UDP 真人房间透明转发；TCP 0x0065 阶段及成员校验；放泡/爆炸具体 wire schema；死亡可靠镜像记账与普通淘汰结算路径；PVE 仲裁者存在。

**需要进一步验证**：
1. 客户端二进制：谁发送 0x0FA3、0x0FA4、0x0FA7；从键盘输入到封包的调用栈，以及远端收到这三类消息后的调用栈。
2. 两客户端本地测试：录制 A/B 的 UDP Type-2 和 TCP 0x0065 报文，按 `GameID, PlayerID, packetNumber, batchIndex, Schema, ClientTime` 关联；验证事件是否重复上传、何时广播。
3. 仲裁切换实验：仲裁客户端在糖泡尚未爆炸时断线，观察新仲裁者是否能恢复计时/爆炸结果。
4. 丢包及乱序实验：丢弃 0x0FA4 或打乱 0x0FA3/0x0FA4 到达次序，确认客户端是否修复状态。
5. 实机验证火焰命中判定方，及 `0x0FA7` 各字段与发送者授权关系。

所有实验需要保留抓包、解析日志和对应代码/客户端逆向地址，避免把服务端重建假设写成原版事实。
