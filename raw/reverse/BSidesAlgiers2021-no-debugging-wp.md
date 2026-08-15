# No Debugging

## 题目简述

附件是一个 stripped、静态链接的 amd64 ELF，没有 PIE，并包含可执行栈和 RWX 段。程序读取最多 32 字节输入，但真正校验过程被拆成两段运行时解密 shellcode，同时加入 ptrace、TracerPid、代码页 CRC 和代码页 XOR 自检。

仓库没有官方 solution README，不过保留了 NASM 源码、反调试 shellcode 源码和生成后的二进制，因此可以在不动态调试目标的情况下完整还原。

## 解题过程

入口先计算自身代码页的 CRC32 与逐字节 XOR，分别保存或和常量比较。随后 mmap 一页 RWX 内存，把第一段字节数组逐字节 XOR 0x3a 后执行。

这段 shellcode 打开 /proc/self/status，查找 TracerPid；若 PID 非零，会向跟踪进程发送信号 14。返回值再与第二段 21 字节 shellcode 的结果组合；后者实际执行 ptrace(PTRACE_TRACEME, 0, 0, 0)。因此直接挂调试器会同时改变返回状态并可能让调试器收到信号。

真正的 306 字节校验器使用前缀 XOR 解码，而不是每字节使用固定密钥：

~~~python
accumulator = 0
decoded = bytearray()

for value in encoded_real_shellcode:
    accumulator ^= value
    decoded.append(accumulator)
~~~

把 decoded 保存后用 64 位模式反汇编，可以看到它再次验证代码 CRC，然后使用 x87 浮点栈和整数运算检查 input 的前三个 32 位字。关键比较包括：

~~~text
CRC 返回值 == 0xfbcae533
input[4:8] + text_crc == 0x234f27c9
其余两个 32 位输入参与两组 x87 常量比较
~~~

成功分支先输出固定前缀“Correct input! the flag is: shellmates{”，再从 input 地址写出恰好 12 字节，最后输出“}\n”。所以需要求解的是 12 字节 flag body，而不是把完整 shellmates 包装送入程序。

对解码后的三个约束求解，并与仓库 NASM 源码中的原始标记交叉核对，得到输入：

~~~text
aghhhhhh_x86
~~~

因此完整 flag 为：

~~~text
shellmates{aghhhhhh_x86}
~~~

## 方法总结

反调试程序不一定需要先对抗调试器。若运行时解密循环和字节数组都在文件中，静态复现解码通常更稳。本题的证据链是“固定 XOR 解出反跟踪逻辑 → 前缀 XOR 解出真实校验器 → 将 x87/整数比较转成输入约束 → 用成功分支的 12 字节写出长度验证候选”。分析自修改 shellcode 时，必须区分固定 XOR 与累计 XOR，否则从第二个字节开始就会全部反汇编错位。
