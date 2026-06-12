# bytezoo WP

## 题目简述

题目给了一个受限执行环境，需要提交一段 proof-of-work/shellcode。shellcode 需要在强约束下先恢复可执行栈或等价执行环境，再发送第二阶段代码读取 `/flag`。

预期解法是利用 `memmove` 与 `mprotect`：先把可执行页一路移动到栈所在页附近，再把栈页改成可执行，最后跳到第二阶段 shellcode 读取 `/flag`。

## 解题过程

第一阶段 shellcode 的目标不是直接读 flag，而是先构造执行环境。原题限制导致很多常规指令或系统调用参数不能直接写出，因此脚本中通过寄存器交换、循环和算术构造出页大小 `0x1000`、目标地址和权限参数。

整体流程如下：

1. 取得当前可执行段地址，并计算它到栈页之间的距离。
2. 如果距离关系满足条件，走 `memmove` 路径，把可执行页逐步搬到栈页附近。
3. 构造 `mprotect(stack_page, 0x1000, PROT_READ | PROT_WRITE | PROT_EXEC)`。
4. 栈页变成可执行后，发送第二阶段 shellcode。
5. 第二阶段执行 `open("/flag", 0)`、`read(fd, buf, 0x100)`、`write(1, buf, n)` 读出 flag。

官方 WP 中还提到，出题时 `fs_base` 的情况超出预期，导致 revenge 版本发布时出现了一些混淆。另有非预期做法可以借助 `sigaction` 达成类似效果。

## 方法总结

这题的重点在于把受限 shellcode 转化成“改内存权限后执行二阶段”的通用模型。第一阶段只负责移动/映射/改权限，第二阶段再使用普通 syscall 读 flag。遇到类似题时，应先判断是否能通过 `mprotect` 或信号机制把受限环境切回可控执行流。
