# UMDCTF 2018 - Proto-Tough

## 题目简述

题目提供 `challenge.proto` 和一个 TCP 服务。服务要求客户端发送合法的 Protocol Buffers 请求，随后把 flag 放在另一个 Protobuf 消息中返回。

## 解题过程

协议定义非常短：

```protobuf
syntax = "proto2";

message get_flag {
}

message send_flag {
    required string flag = 1;
}
```

`get_flag` 没有字段，所以它的合法序列化结果就是空字节串。服务器源码还要求数据以 7 字节魔数 `UMDCSEC` 开头。因此请求只需发送：

```python
import socket
import challenge_pb2

sock = socket.create_connection(("target", 12344))
request = challenge_pb2.get_flag()
sock.sendall(b"UMDCSEC" + request.SerializeToString())
```

响应不是裸文本，而是 `send_flag` 消息。按同一份 `.proto` 生成绑定并解析：

```python
data = sock.recv(1024)
response = challenge_pb2.send_flag()
response.ParseFromString(data)
print(response.flag)
```

结果为：

```text
UMDCSEC-{n0W_y0Ur_tH1nKInG_W1th_pR0toBufs!}
```

该题前缀确实是 `UMDCSEC`。去掉服务器附带的换行后计算 SHA-256，与 `README.md` 的摘要一致。

## 方法总结

空 Protobuf 消息仍是合法消息，其编码长度可以为零。面对自定义 TCP 协议时，应把应用层魔数与消息体分别处理，并使用 `.proto` 生成的解析器读取响应，避免把二进制字段标签误当成 flag 的一部分。
