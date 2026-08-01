# BYUCTF 2023 - RevEng

## 题目简述

附件是未剥离符号的 x86-64 ELF，运行时要求正确 passphrase。符号表中直接保留了 `check_passphrase`，该函数在调用 `strcmp` 前准备用户输入和正确字符串。

## 解题过程

用 `objdump -d` 或 GDB 定位 `check_passphrase`，在 `strcmp@plt` 前断下。System V AMD64 调用约定中前两个参数位于 `rdi`、`rsi`；官方截图所用编译结果还可从相关临时寄存器追踪两个指针。查看字符串：

```gdb
x/s $rdi
x/s $rsi
```

其中一个是输入，另一个为：

```text
She turned me into a newt
```

重新运行并输入该句，程序输出：

```text
byuctf{i_G0t_3etTeR!_1975}
```

本地实际运行还会在 flag 后打印少量非文本尾字节，这是程序字符串终止处理的问题，不属于 flag。

## 方法总结

未剥离符号会大幅降低逆向成本。看到 `strcmp`、`memcmp` 等比较函数时，优先按 ABI 检查调用参数；这比从成功分支向前手算全部汇编更直接。
