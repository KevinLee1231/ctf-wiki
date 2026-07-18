# week2Gandalf's guidance

## 题目简述

服务端给出 8 字符后缀 `suffix` 与 SHA-256 摘要，要求找出 4 字符前缀 `XXXX`，使

$$
\operatorname{SHA256}(\text{XXXX}\mathbin\Vert\text{suffix})=\text{target}.
$$

通过 PoW 后服务端直接返回 flag。

## 解题过程

SHA-256 不需要也无法按通常方式逆运算，但未知量只有 4 个字符，字符集是大小写字母和数字，共 $62^4$ 种组合，直接穷举即可。下面保留完整交互逻辑，并把已失效的历史地址改为运行时填写：

```python
from hashlib import sha256
from itertools import product
import string
from pwn import remote

HOST = "challenge.example"
PORT = 0
alphabet = string.ascii_letters + string.digits

io = remote(HOST, PORT)
io.recvuntil(b"sha256(XXXX+")
suffix = io.recvuntil(b")", drop=True)
io.recvuntil(b"== ")
target = io.recvline().strip().decode()

prefix = None
for chars in product(alphabet, repeat=4):
    candidate = "".join(chars).encode()
    if sha256(candidate + suffix).hexdigest() == target:
        prefix = candidate
        break

if prefix is None:
    raise ValueError("PoW 无解")

io.sendlineafter(b"Plz Tell Me XXXX :", prefix)
print(io.recvall().decode())
```

仓库服务端使用的 flag 为：

```text
0xGame{y0u_p@ss_proof_0f_w0rk}
```

## 方法总结

PoW 的目标是让客户端付出可控计算成本。判断是否可爆破要看未知空间而不是哈希位数；本题只需遍历 $62^4$ 个前缀，并在命中后立即退出循环。
