# Mysterious Vault

## 题目简述

主程序把用户输入的 passcode 复制到固定地址 `0x1337000` 的 System V 共享内存，随后 fork 两个静态链接的 password checker。两个 checker 分别在不同的 chroot 中运行，因此各自只能读到其目录内名为 `password` 的秘密；只有二者都正常退出且共享内存前 `0x40` 字节等于主程序内置的正确短语时，vault 才会打印 flag。

checker 在关闭标准输入、输出和错误后才安装 seccomp。过滤器仅允许 `read`、`write`、`open` 与 `exit`，但这不足以阻止已经控制返回地址的 ROP 链。真正漏洞在于 `memcpy(password, shared + 8, passwd_len)`：目的栈缓冲区只有 200 字节，`passwd_len` 由主程序接收的最多 `0x1ff` 字节输入决定。两个 checker 均无 canary、无 PIE 且静态链接，具备大量 ROP gadget。

## 解题过程

### 绕过字符串检查并同时溢出两个 checker

checker 溢出后会执行：

```c
if (strcspn(password, "password") != strlen(password)) {
    exit(1);
}
```

输入是二进制而不是 C 字符串，因此把首字节置为 `\0` 即可令 `strcspn` 与 `strlen` 都在开头停止。`memcpy` 仍会复制完整 `passwd_len`，故后面的 ROP payload 不受影响。

同一份共享内存 payload 会进入两个二进制；二者的栈帧大小相差 `0x10`，返回地址落点不同。官方解法没有为两者分别发送 payload，而是利用同一绝对地址在两个静态二进制中对应不同 gadget 的事实分流：在共同偏移处，`0x427ae6` 在 3000 中是 `pop rbx; pop rbp; ret`，在 3001 中是 `add rsp, 0x70; pop rbp; ret`。因此 3001 会跨过仅供 3000 使用的一段链，随后执行自己的链。

### 在 seccomp 白名单内取回两个 password

对 3000，ROP 以 `open("password", O_RDONLY)` 打开 chroot 内文件，再调用 `read(fd, 0x1337000, count)` 把内容写回共享内存，最后进行 `exit(0)`。官方链使用 `xor esi, esi; call rbp` 设置 `O_RDONLY`；`rbp` 指向 `syscall; pop rbx; ret`，其中 `pop rbx` 吸收 `call` 压入的返回地址。

3001 的 gadget 序列不同，额外先选择满足 `rbp-0x28` 可写要求的值；随后同样执行 `open`、`read`，但将自己的结果写入 `0x1337000 + 0x20`。这既避免两个结果互相覆盖，又满足主程序对共享内存拼接短语的比较。两个子进程必须显式 `exit(0)`，因为父进程会将任何非零退出或信号终止视为认证失败。

以下片段保留了分流链的关键布局，省略了仅与该静态构建地址有关的 gadget 常数：

```python
payload = flat({
    0x00: b"\x00",                 # 绕过 strcspn
    0xd8: p64(pop_rdi_3001),
    0xe0: p64(password_3001),
    0xe8: p64(split_gadget),        # 3000: pop/pop/ret；3001: add rsp, 0x70
    # 3000: open -> read(SHARED_ADDR) -> exit(0)
    # 3001: open -> read(SHARED_ADDR + 0x20) -> exit(0)
})
```

### 验证

官方 `solve.py` 先以 `0x1ff` 个 `A` 填充 username 输入，再在 passcode 提示处发送上述二进制 payload；后者的长度足以触发 checker 的栈复制越界。成功条件不是单个 shell，而是两个 checker 都从各自 chroot 读回 password、正常退出，并使父进程打印 `ACCESS GRANTED` 后调用 `get_flag()`。本文只根据官方 `WRITEUP.md`、solver 与源代码核对，没有运行服务或 ROP。

## 方法总结

- 核心技巧：二进制 NUL 绕过基于 C 字符串的过滤，随后用同一溢出数据在相近静态二进制中做栈偏移分流。
- 识别信号：共享内存输入被多个子进程复制、子进程 chroot 保存互补秘密、seccomp 只保留基础文件 syscall 时，应同时检查栈溢出和父进程的退出状态校验。
- 复用要点：多进程利用不能只让其中一个链成功；共享数据的写入偏移、每个子进程的栈对齐和 `exit(0)` 都是满足父进程最终验证的必要条件。
