# CPSC233 Assignment 3

## 题目简述

程序只允许首阶段 shellcode 占 16 字节，无法直接容纳完整 execve("/bin/sh")。但执行环境保留了指向可写可执行区域的寄存器状态，因此可以用短 read shellcode 再读入第二阶段。

## 解题过程

观察进入 shellcode 前的寄存器，rdx 指向后续可用缓冲区。首阶段只需设置 read 的调用号、stdin fd 并触发 syscall，让服务继续读取更大的第二阶段：

~~~asm
xor eax, eax
xor edi, edi
mov rsi, rdx
syscall
jmp rsi
~~~

具体编码需控制在 16 字节内，可利用入口时已有的 rdx 长度值，避免重复设置。发送 stage1 后立即发送 pwntools 的 shellcraft.sh() 作为 stage2；第二阶段执行 execve 并取得 shell，再读取 flag：

~~~text
maple{r34d_5h3llc0d3_u51n6_5h3llc0d3}
~~~

## 方法总结

严格长度限制常提示分阶段加载。先检查入口寄存器和内存权限，尽量复用现成值；stage1 只负责扩大输入能力，复杂逻辑放到 stage2。交互脚本要保证两个阶段的发送时序，避免第二阶段字节被第一轮解析吞掉。
