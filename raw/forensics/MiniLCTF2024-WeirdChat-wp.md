# miniLCTF 2024 WeirdChat Writeup

## 题目简述

附件 `traffic.pcapng` 记录了运行在明文 HTTP 上的 Matrix 聊天。Matrix 消息默认采用端到端加密，因此抓到登录密码或 HTTP 流量并不等于能直接读到加密房间内容；突破口是聊天后段新建的未加密房间，其中泄露了会话密钥导出文件和导出口令。

## 解题过程

### 定位未加密房间与密钥文件

先在 Wireshark 中限定 Matrix HTTP 请求，重点查看 `/_matrix/` 路径和较大的请求体：

```text
http && http.request.uri contains "/_matrix/"
```

对 `m.room.message`、媒体上传和同步响应做 Follow HTTP Stream。仓库 PCAP 中可独立核对到以下请求与明文消息：

```text
POST /_matrix/media/v3/upload?filename=element-keys.txt
Content-Length: 4610

-----BEGIN MEGOLM SESSION DATA-----
...

after all, it's encrypted with MySuperSecretKey ...!!!
```

因此需要导出上传请求的 4610 字节正文为 `element-keys.txt`，口令为：

```text
MySuperSecretKey
```

不要误把 HTTP Bearer token 或账号密码当成 Megolm 会话密钥；端到端消息的解密材料在这个 Key Export 文件中。

### 解开 Key Export

Element 的导出文件不是裸会话密钥。其正文以 `BEGIN MEGOLM SESSION DATA` 包裹，内部数据使用由口令派生的密钥加密，并带有完整性校验。应依据 [Matrix Client-Server 规范的 Key Export 格式](https://spec.matrix.org/v1.10/client-server-api/#key-exports)解析，或使用支持 Element Key Export 的工具导入：

```text
key file: element-keys.txt
password: MySuperSecretKey
```

解包后得到各房间的 `room_id`、`session_id`、`session_key` 和发送者密钥。将这些字段按房间与会话编号建立映射。

### 解密 Megolm 事件并做 Base32 解码

从 PCAP 的同步响应中提取 `m.room.encrypted` 事件，至少保留：

```json
{
  "room_id": "对应房间 ID",
  "content": {
    "algorithm": "m.megolm.v1.aes-sha2",
    "ciphertext": "事件密文",
    "session_id": "对应会话 ID"
  }
}
```

以相同 `room_id` 和 `session_id` 的 `session_key` 初始化 Olm/Megolm 入站会话，再调用群组消息解密接口。可参考 [matrix-utils 的 `megolm_backup.py`](https://github.com/cyphar/matrix-utils/blob/main/megolm_backup.py)理解密钥备份解包，但本题是本地 Key Export，不是服务端备份；消息解密器也必须把 PCAP 中的 room ID 一并传入并校验事件索引。

解密后的聊天中包含一段 Base32 文本，去除空白、补齐 `=` 后解码：

```python
import base64

encoded = "从解密消息取得的 Base32 字符串"
encoded += "=" * (-len(encoded) % 8)
flag = base64.b32decode(encoded).decode()
print(flag)
```

当前仓库保存了完整 PCAP，本文已经复核密钥文件上传和口令泄露；但没有保存官方解密器的实际输出，且当前环境缺少 Olm 绑定，因此不编造最终 Base32 明文。上述字段链条足以在安装兼容 Olm 库的隔离环境中复现。

## 方法总结

明文传输 Matrix 流量仍不等于消息明文，因为 Megolm 加密位于 HTTP 之上。本题真正的取证链是“未加密房间泄露 Key Export 与口令 → 解包会话密钥 → 按 room/session 匹配加密事件 → Megolm 解密 → Base32 解码”。分析协议流量时必须区分传输层是否加密和应用层是否加密，并优先搜索后段的附件上传、备份与恢复操作。
