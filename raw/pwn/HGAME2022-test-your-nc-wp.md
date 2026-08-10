# test_your_nc

## 题目简述

这是一道入门交互题。远程服务先要求完成一个四字符 SHA-256 Proof of Work，验证通过后直接进入 shell；在 shell 中定位并读取 flag 文件即可。

## 解题过程

连接服务后会收到一个 SHA-256 摘要，题目要求找出长度固定为 4 的可打印字符串，使其哈希等于该摘要。可以使用 pwntools 的 `mbruteforce` 枚举：

```python
import hashlib
import string

from pwn import remote
from pwnlib.util.iters import mbruteforce

io = remote("challenge.example", 10000)

io.recvuntil(b") == ")
target = io.recvline().strip().decode()

proof = mbruteforce(
    lambda value: hashlib.sha256(value.encode()).hexdigest() == target,
    string.printable,
    length=4,
    method="fixed",
)

io.sendlineafter(b"????> ", proof.encode())
io.interactive()
```

这里的主机和端口应替换为题目当前实例。进入交互模式后，先列出目录，再读取实际存在的 flag 文件：

```bash
ls -la
cat flag
```

原题环境在比赛期间曾增加 PoW 以缓解端口扫描，因此直接使用 `nc` 只能看到验证提示；完成上述哈希挑战后才会获得 shell。

## 方法总结

本题的重点是熟悉远程服务交互与简单 PoW。解析服务端给出的目标摘要时应去掉换行，枚举范围、字符串长度和哈希算法必须与提示完全一致。通过验证后再切换到交互模式，按正常 Linux 文件操作定位 flag。
