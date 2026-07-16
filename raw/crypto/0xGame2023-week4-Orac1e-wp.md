# Orac1e

## 题目简述

服务使用随机 AES-128 密钥和 IV，以 CBC 模式加密经过 PKCS#7 填充的 flag，并返回 `IV || ciphertext`。随后它允许用户提交任意 Base64 密文：填充合法时返回 `Data update`，否则返回 `False` 或 `invaild input`。这个一比特差异构成 CBC padding oracle。

## 解题过程

对相邻密文块 $C_{i-1}$、$C_i$，CBC 解密满足：

$$
I_i=D_K(C_i),\qquad P_i=I_i\oplus C_{i-1}.
$$

虽然无法直接计算中间值 $I_i$，但可以修改前一块为 $C'_{i-1}$。若希望末尾出现长度为 $t$ 的合法填充，只需逐字节猜测，使：

$$
I_i[j]\oplus C'_{i-1}[j]=t.
$$

一旦 oracle 返回成功，就有 $I_i[j]=C'_{i-1}[j]\oplus t$；恢复完整个 $I_i$ 后，再与原始 $C_{i-1}$ 异或得到明文块。

下面的脚本包含四字符 PoW、稳定的 oracle 判断以及末字节误报排除。地址使用运行时参数传入：

```python
import hashlib
import itertools
import re
import string
from base64 import b64decode, b64encode

from pwn import args, remote, xor

BLOCK_SIZE = 16


def solve_pow(io):
    line = io.recvline().decode()
    match = re.search(
        r"XXXX\+([A-Za-z0-9]{16})\) == ([0-9a-f]{64})", line
    )
    suffix, target = match.groups()
    alphabet = string.ascii_letters + string.digits
    for chars in itertools.product(alphabet, repeat=4):
        prefix = "".join(chars)
        digest = hashlib.sha256((prefix + suffix).encode()).hexdigest()
        if digest == target:
            io.sendlineafter(b"XXXX: ", prefix.encode())
            return
    raise ValueError("PoW solution not found")


io = remote(args.HOST, int(args.PORT))
solve_pow(io)
io.recvuntil(b"Here are the secret:\n")
ciphertext = b64decode(io.recvline().strip())


def valid_padding(payload):
    io.sendlineafter(b"> ", b64encode(payload))
    return io.recvline().strip() == b"Data update"


def decrypt_block(previous_block, current_block):
    intermediate = [0] * BLOCK_SIZE

    for padding in range(1, BLOCK_SIZE + 1):
        index = BLOCK_SIZE - padding
        crafted = bytearray(BLOCK_SIZE)

        for position in range(index + 1, BLOCK_SIZE):
            crafted[position] = intermediate[position] ^ padding

        for guess in range(256):
            crafted[index] = guess
            if not valid_padding(bytes(crafted) + current_block):
                continue

            # padding=1 时排除“碰巧保留原始多字节填充”的误报。
            if padding == 1:
                probe = bytearray(crafted)
                probe[index - 1] ^= 1
                if not valid_padding(bytes(probe) + current_block):
                    continue

            intermediate[index] = guess ^ padding
            break
        else:
            raise ValueError(f"byte {index} was not recovered")

    return xor(bytes(intermediate), previous_block)


blocks = [
    ciphertext[offset:offset + BLOCK_SIZE]
    for offset in range(0, len(ciphertext), BLOCK_SIZE)
]

plaintext = b"".join(
    decrypt_block(blocks[index - 1], blocks[index])
    for index in range(1, len(blocks))
)

padding = plaintext[-1]
assert 1 <= padding <= BLOCK_SIZE
assert plaintext.endswith(bytes([padding]) * padding)
print(plaintext[:-padding].decode())
```

脚本最终输出：

```text
0xGame{475b1a7c40311c4f21d676d2bc02f459}
```

## 方法总结

Padding oracle 泄露的不是密钥，而是“修改后的明文尾部是否具有合法填充”。从块尾向前控制目标填充值，就能逐字节恢复 $D_K(C_i)$，再与原前一密文块异或得到明文。实现时应消费完整提示符、对响应做 `strip()`，并特别处理 $t=1$ 时原始填充造成的假阳性。
