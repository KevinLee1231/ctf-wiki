# DownUnderCTF 2020 - added protection

## 题目简述

附件是一个 64 位 ELF。程序没有直接保存明文 flag，而是把 130 字节数据按 16 位字处理：先与 `0xbeef` 异或，再减去 `42`（发生低位回绕时等价于减去 `43`），随后用 `mmap` 申请可读、可写、可执行内存，经 `memcpy` 复制后作为函数调用。决定性障碍是恢复解码后的 shellcode，而不是利用内存漏洞。

## 解题过程

反编译后可以把主循环整理为：

```c
for (size_t i = 0; i < size / 2; i++) {
    uint16_t *p = ((uint16_t *)code) + i;
    *p ^= 0xbeef;
    if (*p < 0x2a)
        *p -= 0x2b;
    else
        *p -= 0x2a;
}

uint8_t *ptr = mmap(0, size,
    PROT_EXEC | PROT_WRITE | PROT_READ,
    MAP_ANON | MAP_PRIVATE, -1, 0);
memcpy(ptr, code, size);
((void (*)(void))ptr)();
```

静态重写逆变换可以恢复 shellcode；更直接的方法是在 `memcpy` 返回后、间接调用前下断点。此时目标缓冲区已经是明文机器码。在 GDB 中先定位间接调用，再检查其目标地址：

```gdb
disassemble main
break *main+299
run
x/20i $rdx
```

实际偏移会随编译结果变化，因此应以当前二进制的 `memcpy` 和 `call *reg` 为准。解码后的指令包含多条 `movabs`：

```text
movabs $0x64617b4654435544,%r8
movabs $0x6e456465636e3476,%r9
movabs $0x5364337470797263,%r10
movabs $0x65646f436c6c6568,%r11
movabs $0x662075206e61437d,%r12
movabs $0x2065687420646e69,%r13
movabs $0x2020203f67616c66,%r14
```

x86-64 立即数以小端序进入寄存器。不能把整串十六进制只反转一次，而要将每个 8 字节块分别按字节反序，再按寄存器顺序拼接。前四块得到：

```text
DUCTF{adv4ncedEncrypt3dShellCode
```

继续拼接后为：

```text
DUCTF{adv4ncedEncrypt3dShellCode}Can u find the flag?
```

因此 flag 是：

```text
DUCTF{adv4ncedEncrypt3dShellCode}
```

## 方法总结

本题的关键观察点是“程序要执行密文，就必然在某个时刻生成可执行明文”。对于自解码 shellcode，优先追踪 `mmap`、`memcpy` 与间接调用之间的数据流，并在最后一次变换之后检查内存；读取嵌入的 64 位 ASCII 常量时，还要按机器的小端字节序逐块还原。
