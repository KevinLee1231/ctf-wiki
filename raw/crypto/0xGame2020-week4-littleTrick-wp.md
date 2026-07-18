# week4littleTrick

## 题目简述

服务端生成 2048 位 RSA 模数，flag 固定为 44 字节。用户提交一个 RSA 密文，服务端先用私钥解出 `mask`，再把 flag 的同长度后缀替换为该 `mask`，最后返回替换后明文的裸 RSA 密文。

由于 RSA 未使用随机填充，同一明文总是得到同一密文，可以把服务端响应当作逐字节相等性 oracle。

## 解题过程

设已经恢复了前 $i$ 个字节。选择长度为

$$
44-i-1
$$

的已知 `mask`。服务端构造的明文就变成：

```text
已知前 i 字节 || 未知的第 i+1 字节 || 已知 mask
```

本地枚举该未知字节，使用公开的 $(n,e)$ 加密完整候选；只要本地密文与服务端返回值相同，当前字节就恢复成功。所有候选明文只有 44 字节，远小于 2048 位模数，不会发生模截断。

一个不能忽略的边界是：`long_to_bytes(0)` 仍会得到一个 `\x00` 字节，因此服务端能接受的 mask 最短也是 1 字节。oracle 最多泄露 flag 的前 43 字节，最后一个原始字节始终被 mask 覆盖。题目采用标准 `0xGame{...}` 格式，所以最后一字节可确定为 `}`；原脚本强行循环 44 次会在最后一轮找不到候选。

```python
import re
import string
from hashlib import sha256

from Crypto.Util.number import bytes_to_long
from pwn import args, remote
from pwnlib.util.iters import mbruteforce

HOST = args.HOST or "127.0.0.1"
PORT = int(args.PORT or 10004)
io = remote(HOST, PORT)

E = 65537
FLAG_LENGTH = 44


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


proof_of_work()
io.recvuntil(b"n : ")
n = int(io.recvline().strip(), 16)

known = bytearray()

while len(known) < FLAG_LENGTH - 1:
    mask = b"A" * (FLAG_LENGTH - len(known) - 1)
    assert 1 <= len(mask) < FLAG_LENGTH

    encrypted_mask = pow(bytes_to_long(mask), E, n)
    io.sendlineafter(b"> ", b"1")
    io.sendlineafter(
        b"Your mask (in hex): ",
        format(encrypted_mask, "x").encode(),
    )
    target_cipher = int(io.recvline().strip(), 16)

    for candidate in range(32, 127):
        guess = bytes(known) + bytes([candidate]) + mask
        if pow(bytes_to_long(guess), E, n) == target_cipher:
            known.append(candidate)
            print(bytes(known))
            break
    else:
        raise RuntimeError(f"第 {len(known)} 字节不在可打印 ASCII 范围")

known.append(ord("}"))
assert len(known) == FLAG_LENGTH
print(known.decode())
io.close()
```

## 方法总结

漏洞由“可控后缀替换 + 确定性裸 RSA”共同造成。调整 mask 长度即可让响应中只剩一个未知字节，再进行本地加密比对。实现时必须审计 `long_to_bytes` 的零值行为：该接口无法暴露最后一个原始字节，本题只能依据已知 flag 格式补齐结尾。
