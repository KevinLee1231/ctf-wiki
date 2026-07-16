# calc

## 题目简述

程序在 5 秒闹钟限制内连续生成 100 组 `rand()` 加法题，全部答对后直接执行 `/bin/sh`。人工输入来不及，需要用 pwntools 自动解析算式并立即回传答案。

## 解题过程

每轮输出形如 `a+b=`，其中 `a`、`b` 和 `ans` 都是 C 的 `int`。两个 `rand()` 返回值相加可能超过有符号 32 位范围，目标环境会按 32 位结果比较，因此脚本显式转换为 `int32`，避免 Python 无限精度整数与 C 结果不一致。

```python
import re
from ctypes import c_int32
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/pwn")

for _ in range(100):
    output = io.recvuntil(b"=")
    match = re.search(rb"(-?\d+)\+(-?\d+)=$", output)
    assert match, output
    a, b = map(int, match.groups())
    answer = c_int32(a + b).value
    io.sendline(str(answer).encode())

io.interactive()
```

脚本不要开启逐字节 debug 日志，否则大量 I/O 输出可能耗尽 5 秒。连接比赛服务时，将 `process()` 替换为当前题目提供的主机与端口即可。

## 方法总结

交互自动化题的核心是稳定识别协议边界、处理目标语言的数据宽度并控制通信开销。优先用固定分隔符和正则提取字段，不要依赖脆弱的行号；涉及 C 整数时还应考虑符号、位宽和溢出语义。
