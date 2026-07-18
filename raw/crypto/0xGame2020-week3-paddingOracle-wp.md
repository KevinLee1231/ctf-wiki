# week3paddingOracle

## 题目简述

服务端使用 AES-CBC 加密 flag，并允许用户提交任意 16 字节 IV 和整块密文进行解密。服务端不会返回明文，但会分别返回 `success`、`wrong padding` 或 `pad error`，因此形成了 CBC Padding Oracle。

连接后还需完成一个四字符 SHA-256 proof of work。远程地址不属于题目机制，脚本通过 `HOST`、`PORT` 参数接收当前实例地址。

## 解题过程

对 CBC 的第 $i$ 个密文块 $C_i$，先记中间值

$$
I_i=D_K(C_i).
$$

真实明文块为

$$
P_i=I_i\oplus C_{i-1},
$$

其中第一块的 $C_{i-1}$ 就是原始 IV。服务端允许把单个 $C_i$ 与自定义 IV 一起提交，所以可以逐字节构造自定义 IV，使解密结果末尾依次满足 `01`、`02 02`、`03 03 03`，从 oracle 的成功响应反推出整个 $I_i$。

源码的校验逻辑读取明文最后一个字节作为填充长度，要求它不大于 16，并检查末尾相应字节是否全部相等。这个可区分响应正是攻击成立的原因。

以下脚本完整处理 proof of work、逐块恢复和 PKCS#7 去填充。使用方式示例为 `python solve.py HOST=127.0.0.1 PORT=10003`。

```python
import re
import string
from hashlib import sha256

from pwn import args, context, remote, xor
from pwnlib.util.iters import mbruteforce

context.log_level = "info"
HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10003)
io = remote(HOST, PORT)


def proof_of_work():
    line = io.recvline_contains(b"sha256(XXXX+").decode().strip()
    match = re.fullmatch(
        r"sha256\(XXXX\+(.+)\) == ([0-9a-f]{64})",
        line,
    )
    if match is None:
        raise ValueError(f"无法解析 PoW：{line!r}")

    suffix, target = match.groups()
    alphabet = string.ascii_letters + string.digits
    prefix = mbruteforce(
        lambda value: sha256((value + suffix).encode()).hexdigest() == target,
        alphabet,
        4,
        method="fixed",
    )
    io.sendlineafter(b"Give me XXXX: ", prefix.encode())


def oracle(test_iv: bytes, block: bytes) -> bool:
    io.sendlineafter(b"> ", b"1")
    io.sendlineafter(b"Your IV (in hex): ", test_iv.hex().encode())
    io.sendlineafter(b"Your cipher (in hex): ", block.hex().encode())
    return io.recvline().strip() == b"success"


def recover_intermediate(block: bytes) -> bytes:
    intermediate = bytearray(16)
    crafted_iv = bytearray(16)

    for pos in range(15, -1, -1):
        pad = 16 - pos

        for index in range(pos + 1, 16):
            crafted_iv[index] = intermediate[index] ^ pad

        for guess in range(256):
            crafted_iv[pos] = guess
            if not oracle(bytes(crafted_iv), block):
                continue

            # pad=1 时可能碰巧命中原有的多字节填充。
            # 翻转填充范围外的前一字节：真正的 01 仍有效，伪阳性会失效。
            if pad == 1:
                check_iv = bytearray(crafted_iv)
                check_iv[pos - 1] ^= 1
                if not oracle(bytes(check_iv), block):
                    continue

            intermediate[pos] = guess ^ pad
            break
        else:
            raise RuntimeError(f"第 {pos} 字节没有找到有效候选")

    return bytes(intermediate)


proof_of_work()
io.recvuntil(b"iv : ")
original_iv = bytes.fromhex(io.recvline().decode().strip())
io.recvuntil(b"crypttext : ")
ciphertext = bytes.fromhex(io.recvline().decode().strip())

blocks = [ciphertext[i:i + 16] for i in range(0, len(ciphertext), 16)]
plaintext = bytearray()
previous = original_iv

for block in blocks:
    intermediate = recover_intermediate(block)
    plaintext.extend(xor(intermediate, previous))
    previous = block

pad = plaintext[-1]
if not 1 <= pad <= 16 or plaintext[-pad:] != bytes([pad]) * pad:
    raise ValueError("恢复出的明文不满足 PKCS#7")

print(bytes(plaintext[:-pad]).decode())
io.close()
```

## 方法总结

CBC 中前一密文块（或 IV）直接控制下一块解密明文的异或量。只要服务端泄露“填充是否合法”这一位信息，就能用至多约 $256\times16$ 次查询恢复一个块的中间值。实现时还要处理 `pad=1` 的伪阳性，并在最后严格校验、移除 PKCS#7 填充。
