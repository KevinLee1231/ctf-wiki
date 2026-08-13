# rat2libzz

## 题目简述

目标是无 libc 的 AArch64 程序，`read(0, buf, 0x100)` 向 64 字节栈缓冲区读入 256 字节。栈上有手工 canary 和函数指针；自定义 `libzz.so` 提供原始 syscall wrapper 及一个能从栈装载寄存器的 `one_gadget`。

## 解题过程

程序非 PIE，`gift` gadget 会从栈加载 `x1`、`x2`、`x4` 并 `blr x4`。第一阶段故意破坏 guard，使分支通过函数指针调用 `_exit`；覆盖该函数指针为 gadget 链：

```text
x1 = flusheverything@GOT
x2 = 8
x4 = write@plt
```

调用 `write(1, GOT, 8)` 泄露 `libzz.so` 内函数实际地址，减去已知偏移得到库基址。随后计算 `one_gadget = libzz_base + 0x44c`。

程序仍在同一 `read` 调用中接收剩余 payload。第二阶段按 `one_gadget` 的栈布局放置：

```text
[sp+80] = 221          # AArch64 execve syscall
[sp+88] = &"/bin/sh"
[sp+96] = 0
return  = one_gadget
```

`one_gadget` 执行 `ldr x8/[x0/x1]` 后 `svc 0`，得到 shell并读取：

```text
grey{d1D_y0u_g3T_4_g0oD_4Rm_w0RkOU7?hehexds}
```

## 方法总结

无 libc 不代表没有可利用动态库；自定义 syscall 库仍提供 GOT 泄漏和执行 gadget。AArch64 ROP 要按 `x0`–`x8` 调用约定和 16 字节栈对齐布置，不能套用 amd64 的寄存器顺序。这里手工 guard 只检查固定值，却把失败路径的函数指针留在同一可覆盖栈帧，反而提供了第一阶段控制流。
