# NepCTF2026 【Game】共生（联机） Writeup

## 题目简述

联机部分仍是 Unity IL2CPP 游戏，但 flag 判定发生在远程房间协议中。客户端正常流程要求两名玩家组队并完成一段运行；反编译 `J07.Network.Core.Protocol` 后可以直接构造 UDP 数据包，伪造完整的建房、认证、开局、心跳和通关流程。

服务端同时提供 HTTP 注册与领奖接口，因此需要两个不同的 team token 创建两套会话，再将它们分别绑定为房主和访客。

## 解题过程

协议使用小端序，UDP 包头固定为：

```text
magic:u16 || type:u8 || player_id:u8 || seq:u32 || body
```

其中 `magic = 0x484A`。关键消息类型如下：

| 类型 | 值 | 作用 |
| --- | ---: | --- |
| `HELLO` | `0x01` | 建立 UDP 会话 |
| `HEARTBEAT` | `0x03` | 保活 |
| `ROOM_CONTROL` | `0x05` | 开局与确认 |
| `CREATE_ROOM` | `0x06` | 创建房间 |
| `JOIN_ROOM` | `0x07` | 加入房间 |
| `AUTH_BIND` | `0x08` | 绑定 HTTP 凭据 |
| `RUN_COMPLETE` | `0x09` | 宣告通关 |

对应的主要响应为 `WELCOME=0x10`、`PEER_JOINED=0x11`、`HEARTBEAT_ACK=0x14`、`RELAY_ROOM_CONTROL=0x17`、`ROOM_RESULT=0x18`、`AUTH_RESULT=0x19` 和 `RUN_COMPLETE_RESULT=0x1A`。

基础封包函数为：

```python
from pwn import *

context.endian = "little"
MAGIC = 0x484A

def pkt(msg_type, player_id, seq, body=b""):
    return (
        p16(MAGIC) + p8(msg_type) + p8(player_id) +
        p32(seq) + body
    )
```

完整流程如下：

1. 分别向 `/api/v1/register` 提交两个不同的 `team_token`，取得各自的 `session_id` 和 `credential`；
2. 两个 UDP socket 分别发送 `HELLO`；
3. 房主发送 `CREATE_ROOM`，访客使用返回的 `room_code` 发送 `JOIN_ROOM`；
4. 将 `session_id || credential` 作为 `AUTH_BIND` 的 body，分别绑定玩家 0 和玩家 1；
5. 房主发送 `ROOM_START_GAME=2`，访客收到转发后发送 `ROOM_START_ACK=3`；
6. 保持连接至少约 61 秒，期间每 5 秒向两端发送心跳；
7. 房主发送 `RUN_COMPLETE`，等待两端返回成功；
8. 使用已注册会话调用 `/api/v1/claim` 领取 flag。

开局控制体为：

```python
start_seq = 0xC0FFEE

host.send(pkt(
    0x05, 0, 4,
    p32(start_seq) + p32(1) + p8(2) + p8(0) + p16(0)
))

guest.send(pkt(
    0x05, 1, 4,
    p32(start_seq) + p32(2) + p8(3) + p8(0) + p16(0)
))
```

通关体只需要相同的 `start_sequence` 和新的完成序号：

```python
host.send(pkt(0x09, 0, 100, p32(start_seq) + p32(1)))
```

若返回码为 `5`，含义是 `RUN_COMPLETE_TOO_EARLY`，说明等待时间不足；增加等待时间并继续发送心跳即可。成功后调用：

```json
{
  "session_id": "<注册返回值>",
  "credential": "<注册返回值>"
}
```

向 `/api/v1/claim` 请求，响应中的 `flag` 即为答案。

## 方法总结

联机部分的关键是从 IL2CPP 元数据中恢复数据包结构和状态转换，而不是攻击 Unity 渲染层。协议没有为通关事件提供不可伪造的游戏过程证明，只检查房间状态、绑定身份和最短运行时间，因此合法构造协议状态即可通过。实现时要保证两个 token 不同、两端的 `room_session` 一致，并在等待窗口内持续心跳。
