# bi0sCTF 2022 - kawaii_vm

## 题目简述

题目是一个 64 位字节码虚拟机。它拥有 4 个通用寄存器、程序计数器、栈指针和数组指针；字节码区、VM 栈和数组区各预留 `0x10000` 字节。程序启用 PIE、NX、Canary、Full RELRO，并加载 seccomp，仅允许 `open/openat/read/write/exit/exit_group`，所以最终利用需要构造 ORW ROP，而不是直接 `execve`。

VM 在读入字节码前允许用户以浮点数指定数组页数。漏洞是用普通大小比较检查浮点数，却没有拒绝 NaN。

## 解题过程

### 用 NaN 绕过数组边界

关键代码为：

```c
scanf("%f", &units);
if (units < 0 || units > MAX_ARRAY_SZ / 0x1000)
    error("Array size isn't kawaii :/");
array_size = 0x1000 * units;
```

IEEE-754 的 NaN 与任何普通数做 `<` 或 `>` 比较都返回 false，因此输入 `NaN` 能同时绕过上下界。该浮点值转成无符号整数时，在目标构建环境中形成高位异常值，使 `array_size / 4` 变成极大范围。字节码检查器随后认可远超真实数组的索引：

```c
if (reg >= MAX_REGS || index >= array_size / 4)
    invalid();
```

实际 `mmap` 仍然只有 `BYTECODE_SZ + STACK_SZ + MAX_ARRAY_SZ`，而 `GET/SET` 以 `regs->ar` 为基址读写 32 位值，于是获得跨越 VM 映射边界的 OOB 读写。

### 建立地址泄漏并伪造 fastbin

利用官方脚本中的固定相对偏移，使用两个相邻的 32 位 `GET` 拼成 64 位指针，分别泄漏：

- `environ` 指向的真实用户栈；
- libc/ld 可写区指针，用于计算 libc 基址；
- ELF 指针，用于计算主程序基址。

先把这些地址拆成高低 32 位保存回数组，避免后续堆布局变化破坏结果。然后在数组区伪造一个大小为 `0x40` 的 fastbin chunk，并通过 OOB `SET` 覆盖 `main_arena` 中相应 fastbin 的 `fd`。目标环境使用 glibc 2.36，必须按 safe-linking 规则编码链指针：

$$
fd_{stored}=fd_{target}\oplus(chunk_{address}\mathbin{\gg}12).
$$

### RESET 让寄存器上下文落到伪 chunk

`RESET` 指令会再次调用 `init_vm()`：

```c
case RESET:
    saved_pc = regs->pc;
    init_vm();
    regs->pc = saved_pc;
    memset(kawaii_map + BYTECODE_SZ, 0, STACK_SZ);
    break;
```

旧的寄存器对象没有释放，新一次 `malloc(sizeof(kawaii_registers))` 从已经污染的 fastbin 取得伪 chunk，于是 `regs` 被分配到受控数组中。之后可以直接改写寄存器上下文里的 `sp`，令其指向泄漏出的真实栈返回地址附近。

### 用 VM PUSH 写入 ORW ROP

VM 的 `PUSH` 会先递减 `regs->sp`，再写入一个 64 位寄存器值。把 `sp` 指向真实栈后，连续 `PUSH` 就等价于受控的向下栈写。官方字节码先用 `MOV/MUL/ADD` 拼出 64 位地址，再依次压入：

```text
open("flag.txt", 0)
read(fd, bss, 0x40)
puts(bss)
```

实际链使用 libc 中的 `pop rdi`、`pop rsi`、`pop rdx; pop rbx`、`open`、`read` 和 `puts`。最后执行 `HALT`，让解释器返回并取走已被覆盖的返回地址，ROP 链随即执行。输入流程的关键部分为：

```python
sendlineafter(b"> ", b"y")
sendlineafter(b"> ", b"NaN")
sendafter(b"> ", assembled_bytecode)
```

输出 flag：

```text
bi0sctf{kawaii_vm_n0t_s0_k4wa1i_4ft3r_4ll_f97cf315ea3a}
```

seccomp、safe-linking 和各阶段内存布局可对照 [官方赛后题解](https://blog.bi0s.in/2023/01/25/Pwn/bi0sCTF22-kawaii_vm/)；正文已经保留了复现所需的 NaN 条件、伪 chunk 编码、`RESET` 重分配和 ORW 链。

## 方法总结

本题把浮点语义漏洞转化成 VM 数组 OOB，再用堆分配器把数组控制升级成寄存器上下文控制。审计浮点边界时必须单独考虑 NaN，因为“既不小于下界，也不大于上界”并不代表值合法。利用上的关键 pivot 是 `RESET`：它提供了新的同尺寸堆分配，使被污染的 fastbin 能把核心 VM 状态放到攻击者可写区；之后原本受限的 `PUSH` 指令就成为写真实栈的工具。
