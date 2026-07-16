# EzLogin-II

## 题目简述

菜单的 `F` 功能返回 AES-CBC 加密的 flag2，`L` 功能则解密用户提交的 Cookie。服务端对填充错误和 JSON 解析错误返回不同消息，形成 Padding Oracle；利用该差异可以逐字节恢复 flag2，而无需知道 AES 密钥。

## 解题过程

CBC 解密第 $i$ 块满足：

$$
P_i=D_K(C_i)\oplus C_{i-1}
$$

攻击者固定目标密文块 $C_i$，修改前一块 $C_{i-1}$，即可控制解密明文的异或结果。服务端处理顺序为“Base64 解码 → CBC 解密 → 去填充 → UTF-8 解码 → JSON 解析”，并泄露两类响应：

- 填充错误、解码错误等落入宽泛异常，返回 `[!] Unkown Wrong`；
- 填充有效但 flag 文本不是 JSON，返回 `[!] JSON Wrong`。

因此，“响应不是 `Unkown Wrong`”可以作为填充有效的判据。对每一块从末字节向前枚举，使末尾依次成为 `01`、`02 02`、……、`10` 重复 16 次，就能恢复 $D_K(C_i)$，再与原前块异或得到 $P_i$。

完整脚本如下：

```python
from base64 import b64decode, b64encode
from pwn import remote

HOST = "TARGET"
PORT = 10005
BLOCK_SIZE = 16

io = remote(HOST, PORT)

# 获取 IV || ciphertext。
io.sendlineafter(b">", b"F")
io.recvuntil(b"[+] Here is flag2 : ")
encrypted = b64decode(io.recvline().strip())

# 进入 login 内层循环；失败时服务端会继续接收下一枚 Cookie。
io.sendlineafter(b">", b"L")

def padding_valid(payload: bytes) -> bool:
    io.sendlineafter(b">", b64encode(payload))
    response = io.recvline()
    return b"Unkown Wrong" not in response

def recover_block(previous: bytes, current: bytes) -> bytes:
    intermediate = bytearray(BLOCK_SIZE)
    plaintext = bytearray(BLOCK_SIZE)

    for position in range(BLOCK_SIZE - 1, -1, -1):
        pad = BLOCK_SIZE - position
        crafted = bytearray(previous)

        for index in range(position + 1, BLOCK_SIZE):
            crafted[index] = intermediate[index] ^ pad

        for candidate in range(256):
            crafted[position] = candidate
            if not padding_valid(bytes(crafted) + current):
                continue

            # 排除原密文恰好已经具有多字节合法填充的假阳性。
            if pad == 1 and position > 0:
                probe = bytearray(crafted)
                probe[position - 1] ^= 1
                if not padding_valid(bytes(probe) + current):
                    continue

            intermediate[position] = candidate ^ pad
            plaintext[position] = intermediate[position] ^ previous[position]
            break
        else:
            raise RuntimeError(f"无法恢复字节 {position}")

    return bytes(plaintext)

blocks = [
    encrypted[i:i + BLOCK_SIZE]
    for i in range(0, len(encrypted), BLOCK_SIZE)
]

plaintext = b"".join(
    recover_block(blocks[i - 1], blocks[i])
    for i in range(1, len(blocks))
)

pad = plaintext[-1]
assert 1 <= pad <= BLOCK_SIZE
assert plaintext.endswith(bytes([pad]) * pad)
flag = plaintext[:-pad]
print(flag.decode())
io.close()
```

恢复结果为：

```text
0xGame{6e02937e-634d-4f6f-8ef6-e5f387006cde}
```

## 方法总结

Padding Oracle 的根因不是 CBC 本身可解出密钥，而是服务端泄露了“填充是否正确”这一位信息；重复查询即可逐字节恢复明文。修复时应使用 AEAD，并在任何解密、填充、解码或业务解析失败时返回完全一致的外部响应，同时避免可观察的时序差异。
