---
type: tooling
tags: [pwn, tooling, tools, environment]
skills: [ctf-pwn]
updated: 2026-07-06
---

# Pwn Tooling

本页记录 `ctf-pwn` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 工具选择边界

### 入口选择

- 首轮以 `file`、`checksec`、`gdb/pwndbg`、`pwntools`、`ROPgadget/ropper`、`seccomp-tools` 为主。
- Python exploit 默认在 `ctf-tools` 中执行；需要调试时用 WSL system-global GDB + pwndbg。
- Kernel/system/qemu 题可能长时间运行，启动服务、编译 rootfs 或跑大量 fuzz 前要先确认。

### 不应进入 Pwn 工具链的情况

- 主障碍仍是理解算法、壳、VM、协议或文件格式时，先转 Reverse/Forensics，不急着写 exploit。
- 没有崩溃、越界、格式化输出、任意读写、syscall 过滤或 sandbox primitive 时，不把普通 binary 题归入 Pwn。
- browser/JIT、kernel/QEMU 和沙箱题需要先确认运行环境和最小触发，不直接跑长时间 fuzz。

### 补工具经验的触发条件

- raw 涉及 glibc 2.35+ heap、safe-linking、IO_FILE 变化，并能抽出稳定调试步骤。
- CET/IBT/Shadow Stack 影响控制流劫持，需要记录替代路线和工具限制。
- kernel exploit 题出现 KASLR/KPTI/SMEP/SMAP、userfaultfd 替代或 modprobe_path 变化。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| `file` / `checksec` / `strings` | 最快建立目标画像 |
| `nc` | 远程服务题先确认交互和输入面 |
| `pwntools` | 脚本化交互、偏移、payload 和 remote 流程 |
| GDB + pwndbg | 一旦需要确认崩溃和寄存器状态，就进入主力 |

### 专项按需

- ROP 与 libc：`ROPgadget`、`ropper`、`one_gadget`
- 防护与沙箱：`seccomp-tools`
- 跨架构：`qemu-*`
- 深度分析/自动化：`angr`、`z3-solver`、`capstone`、`keystone-engine`、`unicorn`、`pyelftools`

### 当前未装 / 建议按需补装

- 当前没有明显缺口。PWN 这套已经足够完整，后续更值得优化的是 libc 数据库和题型脚手架，而不是继续堆基础工具。

## 失败信号与转向

- `checksec` 和 crash 信息不足以判断漏洞族：先最小化输入并转 [pwn-first-pass-red-flags-and-protections.md](pwn-first-pass-red-flags-and-protections.md)，不要直接拼 ROP。
- GDB 本地可打通但远程失败：优先核对 libc、PIE base、I/O 同步、超时和环境变量；若是保护组合问题，转 [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)。
- `one_gadget`、ROPgadget 或 ropper 无可用链：回到 primitive 质量，判断是否需要 leak、栈迁移、ret2dlresolve、SROP、FSOP 或 heap 路线。
- kernel/QEMU/namespace 题跑不稳：先记录启动脚本、initramfs、保护位和调试改动，再转 [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md) 或 [kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md)。

## 详细清单

### Python 包（ctf-tools conda 环境）

| 工具 | 版本 | 功能 | 典型用法 |
|---|---|---|---|
| **pwntools** | 4.15.0 | PWN 全栈框架（远程连接、ROP 链、shellcode、checksec、cyclic、DynELF、corefile） | `from pwn import *`；`pwn checksec binary`；`pwn cyclic 200` |
| **ROPgadget** | 7.7 | ROP gadget 搜索，支持 x86/x64/ARM/MIPS/PowerPC/SPARC | `ROPgadget --binary ./bin \| grep "pop rdi"` |
| **ropper** | 1.13.13 | 替代性 ROP gadget 搜索，支持语义搜索与链式查询 | `ropper -f ./bin --search "pop rdi; ret"` |
| **capstone** | 5.0.6 | 轻量级多架构反汇编引擎 | `from capstone import *; Cs(CS_ARCH_X86, CS_MODE_64)` |
| **keystone-engine** | 0.9.2 | 轻量级多架构汇编引擎，运行时生成 shellcode | `from keystone import *; Ks(KS_ARCH_X86, KS_MODE_64)` |
| **unicorn** | 2.1.2 | CPU 模拟引擎，模拟执行 shellcode 或分析混淆代码 | `from unicorn import *; Uc(UC_ARCH_X86, UC_MODE_64)` |
| **angr** | 9.2.209 | 二进制符号执行框架，自动漏洞发现/约束求解/CFG 恢复 | `import angr; angr.Project('./bin')` |
| **z3-solver** | 4.13.0 | SMT 约束求解器，独立路径条件求解 | `from z3 import *; s = Solver()` |
| **pyelftools** | 0.32 | ELF 文件解析，读取节头/段头/动态符号 | `from elftools.elf.elffile import ELFFile` |
| **numpy** | 2.4.4 | 数组/矩阵运算，权重/偏置操作 | `import numpy as np` |

```bash
conda activate ctf-tools
pwn checksec <binary>              # 查 PIE/RELRO/NX/Canary
pwn cyclic 200 / pwn cyclic -l <v> # De Bruijn 偏移计算
pwn shellcraft amd64.linux.sh()    # shellcode 生成
conda deactivate
```

### 系统全局命令（WSL Kali）

#### 调试与分析

| 工具 | 路径 | 版本 | 功能 | 典型用法 |
|---|---|---|---|---|
| **GDB + pwndbg** | `/usr/bin/gdb` + `/home/kali/pwndbg` | 17.1 | 源码/汇编级调试，`~/.gdbinit` 已配置自动加载 pwndbg | `gdb -q ./bin -ex 'start' -ex 'checksec'` |
| **objdump** | `/usr/bin/objdump` | 2.46 | 二进制反汇编 | `objdump -d ./bin \| grep -B1 "pop.*rdi"` |
| **readelf** | `/usr/bin/readelf` | 2.46 | ELF 结构分析（节头/段头/符号表/动态表） | `readelf -h ./bin; readelf -S ./bin` |
| **nm** | `/usr/bin/nm` | 2.46 | 符号表列出 | `nm ./bin \| grep win` |
| **strings** | `/usr/bin/strings` | 2.46 | 提取可读字符串（GLIBC 版本检测/密码搜索） | `strings libc.so.6 \| grep GLIBC` |
| **file** | `/usr/bin/file` | 5.46 | 识别文件类型（位宽/架构/链接方式） | `file ./binary` |
| **ldd** | `/usr/bin/ldd` | 2.42 | 显示共享库依赖与加载路径 | `ldd ./binary` |
| **strace** | `/usr/bin/strace` | 6.18 | 系统调用跟踪 | `strace ./binary` |
| **ltrace** | `/usr/bin/ltrace` | 0.7.91 | 库函数调用跟踪 | `ltrace ./binary` |
| **checksec** | `/usr/bin/checksec` | 2.6.0 | 二进制保护检测（PIE/RELRO/NX/Canary） | `checksec --file=./binary` |

#### ROP / Gadget / Seccomp

| 工具 | 路径 | 版本 | 功能 | 典型用法 |
|---|---|---|---|---|
| **one_gadget** | `/usr/local/bin/one_gadget` | 1.10.0 | libc one-shot RCE gadget 查找 | `one_gadget libc.so.6` |
| **seccomp-tools** | `/usr/local/bin/seccomp-tools` | 1.6.2 | dump seccomp BPF 过滤规则 | `seccomp-tools dump ./binary` |

#### 跨架构模拟（qemu-user 10.2.2）

| 工具 | 功能 | 典型用法 |
|---|---|---|
| **qemu-aarch64** | ARM64 用户态模拟 | `qemu-aarch64 ./arm64_bin` |
| **qemu-arm** | ARM32 用户态模拟（可配合 gdb-multiarch） | `qemu-arm -g 1234 ./arm_bin` |
| **qemu-mips** | MIPS 大端用户态模拟 | `qemu-mips ./mips_bin` |
| **qemu-i386** | x86 32 位用户态模拟 | `qemu-i386 ./x86_32_bin` |
| **qemu-x86_64** | x86-64 用户态模拟 | `qemu-x86_64 ./x86_64_bin` |
| **qemu-system-x86_64** | 完整 x86-64 系统模拟（内核调试/bzImage+initramfs） | `qemu-system-x86_64 -kernel bzImage -initrd rootfs.cpio -nographic -s` |

#### 编译与打包

| 工具 | 路径 | 版本 | 功能 | 典型用法 |
|---|---|---|---|---|
| **upx** | `/usr/bin/upx` | 4.2.4 | 可执行文件压缩（内核 exploit 体积压缩） | `upx --best exploit` |
| **libc6-dbg** | apt 已安装 | 2.42-13 | glibc 调试符号 | GDB 中 `b __malloc` / `p __malloc_hook` |


---

## 补充速查
`checksec`, `one_gadget`, `ropper`, `ROPgadget`, `seccomp-tools dump`, `strings libc | grep GLIBC`. See stack-pivots-srop-and-seccomp-rop.md for full command list and pwntools template.
