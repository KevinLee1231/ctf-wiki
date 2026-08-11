# pac shell

## 题目简述

这是 AArch64/PAC 环境的交互程序，`read64` 与 `write64` 直接接受任意地址，天然提供 64 位任意读写；但主循环在调用前执行 `autiza`，未签名函数指针无法直接控制 PC。漏洞在 `help()`：它对 `BUILTINS[i].fptr` 执行 `paciza` 后把已签名的指针打印出来。攻击者可改写 builtin 指针为任意 gadget，再借 help 这个 signing oracle 得到可被 `autiza` 接受的指针。

## 解题过程

### 将 help 变为 PAC signing oracle

`BUILTINS` 有 `help`、`ls`、`read64`、`write64` 四项。`write64` 可把第三项之后的 function pointer 改成目标 gadget；再次调用 `help` 会对该值执行 `paciza` 并打印结果。主循环输入这个已签名地址，`autiza` 验证通过后才间接调用，从而把原有 arbitrary write 转为签名后的控制流转移。

### 设置寄存器与返回链

官方 solver 选择随附 libc 内的：

```text
ldr x0, [sp, #0x60] ; ldp x29, x30, [sp], #0x150 ; ret
```

它用 arbitrary read 从栈区扫描一个已知 `read64` 指针，反推出当前 `sp`，以避免环境变量改变造成的栈位置漂移。然后写入：

- `sp + 0x60`：`/bin/sh`；
- `sp + 8`：`system`，作为 gadget 恢复时的 `x30`；
- `BUILTINS[3].fptr`：上述 `x0` 装载 gadget。

调用 help 获得该 gadget 的 PAC 签名值，并将该值喂给主循环。gadget 令 `x0` 指向 `/bin/sh`，恢复 `x30=system` 后 `ret`，形成 `system("/bin/sh")`。

### 验证

题目配置中的验证 flag 为 `DUCTF{did_you_just_bruteforce_the_pac?:(}`。本文没有启动 AArch64 QEMU 或取得 shell；地址选择、签名流程与 stack scan 由源码和官方 solver 静态复核。

## 方法总结

- 核心技巧：PAC 防止伪造指针，但若存在能够以同一 modifier 为攻击者地址签名的 gadget/oracle，认证会被转化为可利用的服务。
- 识别信号：程序同时暴露任意写、`pac*` 指令和会打印已签名函数指针的辅助功能时，应检查能否复用其签名上下文。
- 复用要点：PAC modifier、地址随机化和 ABI 栈布局决定可用性；官方使用 stack 泄露定位，而不是假设硬编码 stack 地址永久有效。
