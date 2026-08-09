# xavier

## 题目简述

程序是静态链接的 PIE，可控长度使用无符号数并允许栈溢出。利用需要先泄露栈 canary 和 PIE 基址，再构造不依赖 libc 的系统调用 ROP。

## 解题过程

缓冲区为 32 字节。第一次请求 41 字节，使回显越过缓冲区并带出 canary 的后 7 字节；在前面补一个零字节即可还原完整 canary。第二次请求 56 字节泄露保存的代码指针，据已知偏移计算 PIE 基址。

最终 payload 保留 canary，利用静态二进制中的写内存 gadget 把 `//bin/sh\x00` 放入 `.bss`，再设置寄存器并执行 `syscall`：

```text
rax = 59
rdi = address_of_bin_sh
rsi = 0
rdx = 0
syscall
```

`execve("//bin/sh", 0, 0)` 成功后读取：

```text
n00bz{jU5t_4_e3sy_m0de3rn_bufferoverflow}
```

## 方法总结

现代栈溢出常需分阶段：canary 解决完整性检查，代码指针解决 PIE，静态 gadget 则替代 libc。每次泄露都应从实际回显长度重建缺失字节，并在最终链中维持正确栈布局。
