# AES ECB

## 题目简述

服务为每个进程随机生成一个固定的 16 字节密钥，并返回：

```text
Base64(AES-128-ECB(用户输入 || flag))
```

Ruby OpenSSL 会自动使用 PKCS#7 填充。攻击者可以反复选择任意前缀，而同一连接中的密钥和未知后缀始终不变，因此这是典型的 ECB 逐字节后缀恢复 oracle。

## 解题过程

先逐渐增加输入长度并观察密文长度。密文每增加 16 字节跳变一次，可以确定分组大小为 16。再提交多个相同分组，能在密文中看到重复块，确认服务确实使用 ECB。

恢复第 $i$ 个未知字节时，令：

```text
pad_len = 15 - (i mod 16)
```

发送 `A * pad_len` 后，第 $i$ 个未知字节会被推到当前目标块末尾。随后枚举一个候选字节，发送：

```text
A * pad_len || 已恢复字节 || candidate
```

ECB 对相同明文块给出相同密文块，因此只要候选请求的对应密文块与目标块相等，就找到了下一个字节。下面的脚本在同一 TCP 连接中完成恢复；候选限制为可打印 ASCII，避免换行字节破坏服务的逐行输入协议。

```python
#!/usr/bin/env python3
from base64 import b64decode
import sys

from pwn import remote


host = sys.argv[1]
port = int(sys.argv[2])
io = remote(host, port)


def oracle(payload: bytes) -> bytes:
    io.sendlineafter(b"=> ", payload)
    io.recvuntil(b"The result : ")
    return b64decode(io.recvline().strip())


block_size = 16
alphabet = range(0x20, 0x7F)
known = bytearray()

while not known.endswith(b"}"):
    index = len(known)
    prefix = b"A" * (block_size - 1 - index % block_size)
    block = index // block_size
    left = block * block_size
    right = left + block_size

    target = oracle(prefix)[left:right]
    table = {}
    for candidate in alphabet:
        probe = prefix + bytes(known) + bytes([candidate])
        table[oracle(probe)[left:right]] = candidate

    if target not in table:
        raise RuntimeError("没有找到候选字节，请检查连接和字符集")
    known.append(table[target])
    print(known.decode())

io.close()
```

恢复结果为：

```text
shellmates{I_though_AES_w4s_m1l1tary_gr4de_encryp7ion_1n_al1_m0des}
```

## 方法总结

本题利用的不是 AES 算法本身，而是 ECB 的确定性与“可控前缀拼接未知后缀”这一接口设计。看到固定密钥、可重复查询、ECB 和未知后缀同时出现时，应立即尝试按块对齐的字典攻击。

复现时必须保持在同一密钥生命周期内查询，并准确截取第 $\lfloor i/16 \rfloor$ 个密文块。实际服务若限制字符集、请求次数或每次请求重新生成密钥，则需要相应调整，不能机械套用逐字节恢复脚本。
