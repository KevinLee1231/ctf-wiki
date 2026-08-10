# CPSC233 Assignment 1

## 题目简述

服务把用户提交的机器码映射为可执行内存，并把三个整数按 System V AMD64 ABI 传给该函数。目标是编写返回三数之和的最小 x86-64 函数。

## 解题过程

前三个整数参数依次位于 rdi、rsi、rdx，返回值放在 rax。可使用：

~~~asm
add rdi, rsi
add rdi, rdx
mov rax, rdi
ret
~~~

用 pwntools 组装并发送十六进制机器码：

~~~python
from pwn import *
context.arch = "amd64"
sc = asm("add rdi,rsi; add rdi,rdx; mov rax,rdi; ret")
io.sendline(sc.hex().encode())
~~~

服务用多组输入校验结果，全部通过后返回：

~~~text
maple{4_51mp13_c41cu14702}
~~~

## 方法总结

本题核心是调用约定，不是内存破坏。写裸函数前应确认架构、参数寄存器、返回寄存器和 ret。若直接使用 rax 累加也可以，但要避免误以为参数通过栈传入。
