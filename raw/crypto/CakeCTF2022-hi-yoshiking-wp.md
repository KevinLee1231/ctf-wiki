# CakeCTF 2022 hi yoshiking Writeup

## 题目简述

服务使用 AES-256-GCM 生成登录令牌，格式为：

```text
iv:ciphertext:tag
```

令牌明文是 JSON：

```json
{"username":"<name>","is_yoshiking":false}
```

只有当用户名为 `yoshiking` 且 `is_yoshiking` 为 `true` 时才返回 flag。虽然 GCM 本应同时保证机密性和完整性，但服务端允许提交任意长度的认证标签，并通过不同异常泄露了标签前缀是否正确。

## 解题过程

先申请用户名为 `yoshiking` 的合法令牌。已知明文为：

```text
{"username":"yoshiking","is_yoshiking":false}
```

将它改成下面的等长字符串：

```text
{"username":"yoshiking","is_yoshiking": true}
```

GCM 的加密部分本质上是计数器模式。相同 IV 下，密文与明文使用同一密钥流，所以可以直接异或已知明文和目标明文的差分：

$$
C'=C\oplus P\oplus P'.
$$

篡改后的密文不再对应原认证标签，正常情况下应在这里失败。问题在于 Ruby OpenSSL 接口接受截断标签。服务还把认证失败表现为 `CipherError`，而“认证通过但标签不足 16 字节”会进入后续逻辑并抛出另一条错误。这样便形成逐字节标签 oracle。

假设已经恢复了正确标签前缀 $T_0\ldots T_{i-1}$，枚举第 $i$ 个字节并提交长度为 $i+1$ 的标签。只要响应不含 `CipherError`，当前字节就是正确值。重复 16 次即可得到完整标签。

```python
from ptrlib import Socket

sock = Socket("crypto.2022.cakectf.com", 10333)

sock.sendlineafter("> ", "1")
sock.sendlineafter("name: ", "yoshiking")
iv, token, _ = [
    bytes.fromhex(part)
    for part in sock.recvlineafter("token: ").strip().decode().split(":")
]

plain = b'{"username":"yoshiking","is_yoshiking":false}'
target = b'{"username":"yoshiking","is_yoshiking": true}'
delta = bytes(a ^ b for a, b in zip(plain, target))
forged = bytes(a ^ b for a, b in zip(token, delta))

tag_prefix = b""
for i in range(16):
    for candidate in range(256):
        trial = tag_prefix + bytes([candidate])
        sock.sendlineafter("> ", "2")
        sock.sendlineafter(
            "token: ",
            f"{iv.hex()}:{forged.hex()}:{trial.hex()}",
        )
        line = sock.recvline().decode()
        if "CipherError" not in line:
            tag_prefix = trial
            if i == 15:
                print(line)
            break
```

完整标签通过验证后，服务将篡改后的 JSON 解释为 yoshiking 的令牌并输出：

```text
CakeCTF{hi_yoshiking_lets_go_sushi}
```

## 方法总结

这题同时利用了两点：计数器模式密文可按已知明文差分修改，以及认证接口允许短标签并泄露了验证结果。GCM 的安全性依赖标签验证被当作一个不可区分的整体；一旦攻击者能逐字节判断标签前缀，完整 128 位标签就退化成 $16\times256$ 规模的枚举。

修复时应固定要求 16 字节标签，在解析明文或返回任何可区分错误之前完成一次完整、统一失败路径的认证检查。
