# DownUnderCTF 2021 - General Skills Quiz

## 题目简述

服务在 30 秒内连续提出 11 个问题，内容包括进制转换、十六进制与 ASCII、URL 编码、Base64、ROT13 和一个固定比赛名问题。每次连接中的单词和数字会随机生成，不能预先抄答案，但所有转换都是标准可逆表示，因此适合用脚本即时解析。

## 解题过程

各轮所需操作如下：

| 轮次 | 输入形式 | 回答方式 |
|---:|---|---|
| 1 | `1+1` | 固定回答 `2` |
| 2 | 十六进制整数 | `int(value, 0)` 转十进制 |
| 3 | 十六进制字节 | `bytes.fromhex` 转 ASCII |
| 4 | URL 百分号编码 | `urllib.parse.unquote` |
| 5 | Base64 | 解码为 UTF-8 |
| 6 | 普通文本 | 编码为 Base64 |
| 7 | ROT13 文本 | 再做一次 ROT13 还原 |
| 8 | 普通文本 | 做 ROT13 编码 |
| 9 | 二进制整数 | `int(value, 2)` 转十进制 |
| 10 | 十进制整数 | `bin` 转带 `0b` 前缀的二进制 |
| 11 | 最佳 CTF 名称 | 固定回答 `DUCTF` |

下面的脚本直接等待每道题的冒号，再读取同一行剩余内容，因此不依赖随机题值：

```python
import base64
import codecs
import urllib.parse
from pwn import remote

io = remote(HOST, PORT)

def next_value():
    io.recvuntil(b": ")
    return io.recvline().strip().decode()

io.sendlineafter(b"quiz...", b"")

io.sendlineafter(b"1+1=?\n", b"2")
io.sendline(str(int(next_value(), 0)).encode())
io.sendline(bytes.fromhex(next_value()).decode().encode())
io.sendline(urllib.parse.unquote(next_value()).encode())
io.sendline(base64.b64decode(next_value()))
io.sendline(base64.b64encode(next_value().encode()))
io.sendline(codecs.decode(next_value(), "rot_13").encode())
io.sendline(codecs.encode(next_value(), "rot_13").encode())
io.sendline(str(int(next_value(), 2)).encode())
io.sendline(bin(int(next_value())).encode())

io.sendlineafter(b"universe?\n", b"DUCTF")
print(io.recvall().decode())
```

全部回答正确后，服务输出：

```text
DUCTF{you_aced_the_quiz!_have_a_gold_star_champion}
```

## 方法总结

本题把常见表示层转换放在短时限交互中，主要难点是稳定解析而不是密码分析。自动化时应以固定提示符切分动态值，并注意编码 API 的输入输出类型：`b64decode` 返回字节，网络库也发送字节；十进制转二进制时服务要求保留 Python `bin` 产生的 `0b` 前缀。
