# GreyCTF 2023 Fast AF Calculator

## 题目简述

程序先把两个用户可控字节写进固定代码洞，生成 `mov al, a; add al, b; ret`，随后在 `gets` 读取的 256 字节栈缓冲区中发生溢出。二进制无 PIE，但给定 libc 启用 ASLR。通过让 JIT 字节在错位地址形成 `pop rdi`、`pop rsi` gadget，可先泄露 libc，再执行 `system("/bin/sh")`。

## 解题过程

代码洞位于 `0x401428`，最终五个字节为：

```text
b0 <a> 04 <b> c3
```

令 `a=0x5f`、`b=0x5e`，分别是 `pop rdi` 与 `pop rsi` 的操作码。于是 `0x401429` 可作为以 `pop rdi` 开头的 gadget，`0x40142b` 为 `pop rsi; ret`。虽然前一个 gadget 后面还会执行 `add al, 0x5e`，但不影响参数寄存器。

溢出返回地址的偏移为 280，缓冲区开头仍需放置 `pleaseeeeeee\0` 通过后续 `strcmp`。第一阶段 ROP 为：

```text
pop rdi
putchar@GOT
pop rsi
0xdeadbeef
print_stuff
main
```

`print_stuff` 会从 GOT 地址开始逐字节调用 `putchar`，从而泄露 `putchar` 实际地址。减去随题 libc 中的符号偏移得到 libc 基址，然后返回 `main` 再触发一次溢出。

第二阶段不再运行计算器，ROP 设置 `rdi` 为 libc 中的 `/bin/sh`，插入一个 `ret` 对齐栈，再调用 `system`：

```python
chain = [pop_rdi, bin_sh, ret, libc.sym.system]
```

进入 shell 后读取 flag：

```text
grey{budg3t_j1t_c0mp1l3r}
```

## 方法总结

用户可控的 JIT 片段即使只有两个立即数字节，也应从每个字节偏移重新反汇编，因为 x86 变长指令可能形成新 gadget。该题把固定地址 JIT、经典栈溢出和两阶段 ret2libc 串在一起；先泄露再回到主函数能稳定绕过 libc ASLR。
