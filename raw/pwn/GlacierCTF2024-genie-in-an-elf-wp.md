# GlacierCTF 2024 Genie in an ELF

## 题目简述

程序先打印 `/proc/self/maps` 和栈上 `FILE *files[2]` 的地址，然后允许两次“任意地址写 1 字节”。每次操作都新建一个指向 `/proc/self/mem` 的 `FILE *`，`fseek()` 到目标位置并 `fwrite()` 一个字节。

限制比表面更特殊：两个字节并不会在 `fwrite()` 时立刻落入进程内存，而是留在两个 8 KiB stdio 缓冲区中，直到 `main` 返回后由 glibc 的 exit 清理流程按链表顺序 flush。预期利用修改清理路径中的两处 libc 指令，让它从栈上取出已有的 `_start` 地址并重新进入程序；获得可重复 reentry 后，每轮用一个字节维持循环、另一个字节逐步写入 shellcode。

## 解题过程

### 1. 解析泄漏并理解延迟写入

`/proc/self/mem` 允许进程通过文件偏移读写自身地址空间，连只读、可执行映射也能修改。程序的核心宏为：

```c
filp = fopen("/proc/self/mem", "wb");
fseek(filp, pos, SEEK_SET);
fwrite(&dat, 1, 1, filp);
```

但源码没有 `fflush(filp)` 或 `fclose(filp)`。第一次 `fwrite()` 会为普通文件流分配全缓冲区，单字节只进入用户态 buffer。`main` 返回后，glibc 的 `exit` handler 调用 `_IO_cleanup`：

1. `_IO_flush_all` 遍历尚未关闭的 `FILE`，此时两个愿望才真正写入 `/proc/self/mem`；
2. `_IO_unbuffer_all` 释放各流缓冲区；
3. 清理函数返回 `__run_exit_handlers`，随后进入 `_exit`。

所以补丁目标必须位于 flush 之后仍会执行的短路径中。利用脚本从 maps 取 PIE 与 libc 基址，并从：

```text
My card deck stacks 0x...
```

解析栈泄漏，所有后续地址都按“泄漏基址 + 随题版本固定偏移”计算。

### 2. 用两字节补丁制造第一次 reentry

glibc 2.39 的 `_IO_unbuffer_all` 内有：

```asm
9696e: 0f 85 0c ff ff ff    jne 96880
```

把相对位移的第二低字节，也就是 `libc + 0x96971`，改为 `0x0b` 后，跳转目标变为 `libc + 0x87480`，落入 `_IO_proc_open` 中部：

```asm
87480: cmp dword ptr [r13+0x70], -1
87485: jne 87441
...
87456: add rsp, 0x108
8745d: pop rbx
8745e: pop r12
87460: pop r13
87462: pop r14
87464: pop r15
87466: pop rbp
87467: ret
```

清理路径此时的 `r13` 和 canary 恰好满足这段代码的检查。栈上虽然已有 `_start`，但默认 `add rsp,0x108` 仍不够远；第二个补丁把 `libc + 0x87459` 的立即数最低字节从 `0x08` 改为 `0x90`，于是实际执行 `add rsp,0x190`。再经过 6 次 `pop`，`ret` 正好取到栈上的 `_start`，程序重新打印欢迎语和泄漏。

两个 `/proc/self/mem` 流在退出时从后创建者向先创建者 flush；无论具体落盘先后，只要进入 `_IO_unbuffer_all` 前两处指令都已更新，跳板就能生效。

### 3. 稳定后续 reentry

第一次 `ret` 消耗了原始栈上的唯一 `_start`。重新启动后，固定位置出现的是 `_start+0x25`，该位置原本会执行 `hlt`。从新一轮清理栈的 `rsp` 到它相距 `0x1e8`；扣除 epilogue 的 6 个 pop，即应使用：

$$
0x1e8-6\times8=0x1b8.
$$

因此下一轮两个愿望分别执行：

```python
write_byte(libc_base + 0x87459, 0xb8)
write_byte(saved_start_plus_25, 0x80)
```

第一项把 gadget 改成 `add rsp,0x1b8`；第二项把返回地址低字节从 `_start+0x25` 修回 `_start`。PIE 映射按页对齐，脚本又已知 `_start` 的实际地址，所以只改最低字节即可。此后每次 `_start` 重新进入时会因 `pop`、16 字节对齐和两次 `push` 让该栈槽向低地址移动 `0x10`，exploit 每轮同步执行：

```python
saved_start_plus_25 -= 0x10
```

### 4. 每轮写一个 shellcode 字节

每次 reentry 仍只有两个字节写：

- 一个字节把当前栈上的 `_start+0x25` 修成 `_start`，维持下一轮；
- 另一个字节写到 PIE 中 `_start+0x25+i`。

该位置属于可执行映射且正常启动路径不会依赖其原内容。官方 payload 共 22 字节：

```asm
mov rdi, <libc 中 "/bin/sh" 的地址>
mov rax, <libc system 的地址>
call rax
```

地址由已泄漏的 libc 基址和随题 `libc.so.6` 求得。完成 22 轮后，最后再提交两次对地址 0 的无效写；程序不检查 `fseek`/flush 失败，这一轮不会修复栈上 `_start+0x25`。清理 gadget 最终返回到该位置，执行已经注入的 shellcode并获得 shell。读取 `/flag.txt` 得到：

```text
gctf{fr33_at_la5t!!!_1ucky_f0r_u5_th3_g3n1e_i5_a_b1t_0bl1v10u5}
```

原官方 WP 的 7 张图片均为源码或 GDB/终端截图；其中的指令、栈偏移、首次成功 reentry、失败原因和最终输出已逐项转写到正文，因此不再保留截图文件。

## 方法总结

本题的关键不是直接把两个字节变成 ROP，而是先分析 stdio 缓冲使写入发生在什么时间，再在极短的 glibc exit 尾部寻找可用控制流。两个初始补丁把一次性原语变为可重复原语；稳定 reentry 后，再以“一字节续命、一字节载荷”的节奏积累任意代码。偏移高度依赖赛题 glibc，复现时必须使用附件版本并用泄漏计算基址，不能照抄绝对地址。
