# DownUnderCTF 2021 - OTWhat 1

## 题目简述

固件更新页面要求提交 URL 和 4096 位 RSA 签名。程序本来使用 RSA PKCS#1 v1.5 与 SHA3-512，但实际验证路径没有调用标准验签：它对签名做裸 RSA 运算，只取结果最后 64 字节，再用 C 字符串语义的 `strcmp` 与 64 字节摘要比较。

```python
decrypted = pow(signature, key.e, key.n).to_bytes(512, "big")
result = strcmp(hash_digest, decrypted[-64:])
```

自定义 `strcmp` 遇到 `\x00` 就停止。如果两个缓冲区的首字节都是零，它会把两段非空二进制数据当作相同字符串；同时程序完全不检查 PKCS#1 v1.5 padding 和 DigestInfo 结构。

## 解题过程

第一步是构造恶意更新 URL，使其 SHA3-512 摘要首字节为零。保留服务要求的恶意服务器前缀，在路径后加入短随机后缀即可，单次命中概率约为 $1/256$：

```python
from Crypto.Hash import SHA3_512
from secrets import token_bytes

while True:
    url = b"https://EVILCODE/" + token_bytes(2)
    if SHA3_512.new(url).digest()[0] == 0:
        break
```

页面 HTML 注释泄露 RSA 公钥。由于验证器不检查标准签名编码，只需寻找一个恰好为 512 字节的随机整数 $s$，使 $s^e\bmod n$ 的最后 64 字节以零开头：

```python
from secrets import token_bytes

while True:
    forged = token_bytes(512)
    value = pow(int.from_bytes(forged, "big"), key.e, key.n)
    decoded = value.to_bytes(key.size_in_bytes(), "big")
    if decoded[-64] == 0:
        break
```

官方旧说明中的示例一度写成 256 字节，但服务明确检查 `len(signature) == 512`，官方 solver 也实际使用 512 字节；应以源码为准。

提交该 URL 和 Base64 编码的伪造签名后，两边比较的第一个字节均为 `\x00`。`strcmp` 立即返回 0，程序把恶意 URL 视为已通过 OEM 签名验证，输出：

```text
DUCTF{https://wiibrew.org/wiki/Signing_bug#L0L_memcmp=strcmp}
```

## 方法总结

本题组合了两个经典签名验证错误：用字符串函数比较二进制摘要，以及只检查裸 RSA 结果的局部字节而不验证 PKCS#1 v1.5 编码。看到 `strcmp(hash, signature_bytes)`、遇 NUL 截断或手写 RSA 验证时，应同时检查比较长度、padding、DigestInfo、哈希算法标识和完整消息绑定。正确做法是调用成熟库的标准验签接口，并使用固定长度、常量时间的二进制比较。
