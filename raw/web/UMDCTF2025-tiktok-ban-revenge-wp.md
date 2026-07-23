# UMDCTF 2025 - tiktok-ban-revenge

## 题目简述

本题沿用 `tiktok-ban` 的 DNS 转发服务，并把过滤条件改为：

```python
if b'tiktok\x03com' in req.lower():
    print("blocked")
```

题目 flag 暗示预期主题是 DNS 名称大小写不敏感，但仓库实际提供的过滤器已经对原始
报文调用 `lower()`；混合大小写的连续域名仍会被拦截。应以可核验的源码和官方
`solve.py` 为准：仓库版本仍通过 DNS 压缩指针绕过。由于核心是协议表示差异，
本文放入 `_unclassified`。

## 解题过程

DNS 名称比较不区分 ASCII 大小写，但仅把整个原始报文转成小写不是正确的协议解析。
更重要的是，`lower()` 不会把压缩名称展开成连续标签。根据
[RFC 1035](https://www.rfc-editor.org/rfc/rfc1035)，名称后缀可以用最高两位为
`11` 的双字节指针引用报文中的其他偏移。

官方 `solve.py` 与前题使用同样的构造：

```python
import struct

header = struct.pack("!HHHHHH", 1, 0x0100, 1, 0, 0, 0)
question = struct.pack(
    "!7s2sHH",
    b"\x06tiktok",
    b"\xc0\x19",
    16,
    1,
)
suffix = b"\x03com\x00"
packet = header + question + suffix
payload = len(packet).to_bytes(4, "big") + packet
```

指针 `\xc0\x19` 引用偏移 `0x19` 处的 `\x03com\x00`。因此：

- 过滤器看到的仍是分离的 `tiktok`、指针和末尾 `com`，不存在连续
  `tiktok\x03com`；
- `req.lower()` 只改变字母大小写，不改变这些字节的布局；
- `dnsmasq` 按 DNS 语义解压后得到 `tiktok.com`，并返回 TXT 记录。

响应中的 flag 为：

```text
UMDCTF{we_remembered_pointer_compression_but_forgor_about_case_insensitivity_:skull:}
```

## 方法总结

不能根据 flag 文案反推一个与仓库源码不符的解法。当前版本的可复现漏洞仍是压缩
名称绕过；若只使用 `TiKtOk.CoM` 的连续标签，`req.lower()` 会将其规范成被禁
字节串并拦截。根本修复仍应先解析并规范化 DNS 名称，再比较语义域名，而不是修改
原始报文后做子串搜索。
