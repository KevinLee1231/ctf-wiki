# Gaming

## 题目简述

目标是一个 Velocity Minecraft 代理。普通客户端状态查询以及常见库的 full stat 查询都看不到 flag；服务端插件只在收到 GameSpy 4 Query 协议的 basic stat 请求时，把 flag 放进 MOTD/hostname 响应。由于主障碍是专用游戏协议交互，暂归 `_unclassified`。

## 解题过程

协议基于 UDP。先发送握手包：魔数 `fe fd`、类型 `0x09`、4 字节大端会话 ID。服务返回 ASCII challenge token：

```python
packet = b"\xfe\xfd" + b"\x09" + struct.pack(">i", session_id)
sock.sendto(packet, (host, port))
reply = sock.recvfrom(2048)[0]
challenge = struct.pack(">i", int(reply[5:-1]))
```

随后发送类型 `0x00` 的 stat 请求，只附上 challenge：

```python
basic = b"\xfe\xfd" + b"\x00" + struct.pack(">i", session_id) + challenge
sock.sendto(basic, (host, port))
data = sock.recvfrom(2048)[0]
```

区别在于 full stat 会在 challenge 后再追加四个 NUL；本题必须省略它们。响应从偏移 5 开始由 NUL 分隔，首字段即插件修改后的 MOTD：

```python
motd = data[5:].split(b"\x00", 1)[0]
print(motd.decode())
```

输出为 `byuctf{th1s!s@Gr8tgam3}`。

## 方法总结

“更完整”的协议请求不一定覆盖所有服务端分支。本题要根据插件源码识别 basic/full stat 的分流条件，并手工构造少四个 NUL 的 basic 请求；通用库若只实现 full stat，就需要下沉到数据包层。
