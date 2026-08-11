# DownUnderCTF 2023 handshake Writeup

## 题目简述

服务接受 CA 签名的 ECC 证书，与证书公钥执行 ECDH，再把共享秘密、客户端 nonce、服务端 nonce 和公开的 `shared_nonce` 输入 HKDF-SHA256。附件中包含有效的 `admin` 证书，但不含管理员私钥；如果能推导会话密钥，就能解密包含 flag 的管理员欢迎消息。

## 解题过程

服务公开：

$$
S_1=\operatorname{SHA256}(N_{c1}\parallel K\parallel N_{s1}),
$$

其中 $K$ 是未知的 32 字节 ECDH 共享秘密。HKDF-Extract 的内层 HMAC 前缀为 $(\text{salt}\oplus\text{ipad})$。令第一次客户端 nonce 恰好等于这个 64 字节前缀：

```python
salt_block = b"DUCTF-2023" + b"\x00" * 54
client_nonce_1 = bytes(a ^ 0x36 for a in salt_block)
```

于是服务返回的 $S_1$ 正好是 HMAC 内层在处理完：

```text
(salt xor ipad) || shared_secret || server_nonce_1
```

之后的 SHA-256 链值。对 $S_1$ 做长度扩展，构造第二次客户端 nonce为 `server_nonce_1 || padding`。第二次连接返回 $N_{s2}$ 和 $S_2$ 后，再从 $S_1$ 延长 `N_s2 || S_2`，便得到第二次 HKDF-Extract 所需的内层摘要。

```python
import hlextend
from Crypto.Hash import HMAC, SHA256
import struct

# 第一次连接返回 server_nonce_1、shared_nonce_1。
ext = hlextend.new("sha256")
client_nonce_2 = ext.extend(
    b"",
    server_nonce_1,
    64 + 32,
    shared_nonce_1.hex(),
)

# 第二次连接返回 server_nonce_2、shared_nonce_2。
ext = hlextend.new("sha256")
ext.extend(
    server_nonce_2 + shared_nonce_2,
    server_nonce_1,
    64 + 32,
    shared_nonce_1.hex(),
)
inner = bytes.fromhex(ext.hexdigest())

outer_prefix = bytes(a ^ 0x5C for a in salt_block)
prk = SHA256.new(outer_prefix + inner).digest()
derived_key = HMAC.new(prk, struct.pack("B", 1), digestmod=SHA256).digest()[:32]
```

这里的 `hlextend.py` 已随官方解题文件给出，实现的是从已知 SHA-256 链值继续压缩。用 `derived_key` 进行 AES-ECB 解密并去除 PKCS#7 填充，即可读到：

```text
DUCTF{k3y_der1v3d_fr0m_ov3rsh4r3d_mat3r1al}
```

## 方法总结

问题不在 ECDH，而在把可控 nonce、共享秘密和公开摘要重复拼进 HKDF，同时把原始 SHA-256 摘要暴露给客户端。精心选择客户端 nonce 后，公开摘要变成了 HMAC 内层状态，Merkle-Damgård 长度扩展便能补全 HKDF。密钥派生应使用有明确域分离和证明边界的协议结构，不能把中间哈希状态作为调试信息公开。
