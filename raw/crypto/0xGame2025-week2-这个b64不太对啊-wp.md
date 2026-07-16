# 这个b64不太对啊

## 题目简述

服务实现标准 Base64 的 3 字节到 4 个 6 位索引转换，但每个连接都会随机打乱 64 字符字母表。用户可以查询任意字符串的编码结果，随后必须提交当前连接使用的完整字母表。

## 解题过程

对完整三字节块 $b_1,b_2,b_3$，第四个 Base64 索引为：

$$
i_4=b_3\mathbin{\&}0x3f
$$

因此固定前两字节为 `AA`，只改变第三字节，就能从编码结果的第四个字符直接得到 `charset[b3 & 0x3f]`。可打印 ASCII 的 33 到 96 共 64 个连续值，其低 6 位恰好把 $0\ldots63$ 各覆盖一次。

随机字母表保存在连接处理器实例中，所以探测和提交必须在同一 TCP 会话完成：

```python
from pwn import *

if not args.HOST or not args.PORT:
    raise SystemExit('usage: python solve.py HOST=<host> PORT=<port>')
host = args.HOST
port = int(args.PORT)
io = remote(host, port)

charset = [None] * 64
io.sendlineafter(b'Choose an option (1/2): ', b'1')

for value in range(33, 97):
    probe = b'AA' + bytes([value])
    io.sendlineafter(b'> ', probe)
    io.recvuntil(b'Result: ')
    encoded = io.recvline().strip().decode()
    charset[value & 0x3f] = encoded[3]

assert all(charset)
guess = ''.join(charset)
assert len(set(guess)) == 64

io.sendlineafter(b'> ', b'!q')
io.sendlineafter(b'Choose an option (1/2): ', b'2')
io.sendlineafter(b'Your charset guess: ', guess.encode())
print(io.recvall().decode())
```

这里恢复的是“索引到字符”的映射，不能按探测顺序直接拼接；必须用 `value & 0x3f` 放回正确索引。

## 方法总结

- 核心技巧：利用 Base64 第四索引只依赖第三明文字节低 6 位的性质构造映射 oracle。
- 识别信号：编码位拆分保持标准 Base64，只随机置换输出字符表，且允许选择明文查询。
- 复用要点：选取连续 64 个字节覆盖全部低 6 位，保持同一会话，并区分“探测值顺序”与“字母表索引顺序”。
