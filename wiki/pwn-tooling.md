---
type: tooling
tags: [pwn, tooling, tools, environment]
skills: [ctf-pwn]
---

# Pwn Tooling

本页维护已确认利用原语之后的本机工具。尚未理解算法、VM、协议或文件格式时先用 [reverse-tooling.md](reverse-tooling.md)，不要因为附件是 ELF 就直接构造利用。

## 平台与环境

离线 ELF 解析、约束计算、字节处理与局部 CPU 模拟优先使用 Windows venv `D:/文档/新建文件夹/venv/Scripts/python.exe`。Linux 本地进程、ptrace/GDB、seccomp、PTY、目标 loader/libc 和 Linux 用户态运行需要 WSL；这类脚本使用 `ctf-tools`。

`ctf-tools` 位于 `/home/kali/miniforge3/envs/ctf-tools`，Python 3.11.15。当前 42 个发行包支撑 pwntools、Frida 及必要依赖，不是预装所有 solver 库的环境。纯远程协议或数学问题可在 Windows 完成时，不以“使用 pwntools 方便”为由复制整套通用包。

| 能力 | Windows venv | WSL `ctf-tools` |
|---|---|---|
| pwntools 原生工作流 | 不作为默认入口 | `pwntools 4.15.0` |
| gadget 搜索 | `ROPGadget 7.7` | `ROPGadget 7.7`，供现有 Pwn 链使用 |
| 多架构反汇编 | `capstone 5.0.7` | `capstone 5.0.6`，pwntools 依赖 |
| 局部 CPU 模拟 | `unicorn 2.1.4` | `unicorn 2.1.2`，pwntools 依赖 |
| ELF 结构解析 | `pyelftools 0.32` | `pyelftools 0.32`，pwntools 依赖 |
| SMT 约束 | `z3-solver 4.16.0.0` | 未安装，独立约束脚本放 Windows |

Frida 的 Linux 插桩入口见 [reverse-tooling.md](reverse-tooling.md)。当前 `ctf-tools` 没有 angr、Qiling、ropper 或 keystone 独立工具入口；需要额外能力时先评估 Windows 新版本及最小实现，不能照旧清单调用或补装。

## 从 pwsh 调用

题目目录在 Windows 为 `D:/题目路径` 时，WSL 对应 `/mnt/d/题目路径`。系统 CLI 直接调用，Conda 使用固定入口，不激活环境：

```pwsh
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/analyze_constraints.py"
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/文档/新建文件夹/venv/Scripts/ROPgadget" --binary "D:/题目路径/binary" --only "pop|ret"
wsl -d kali-linux --exec /usr/bin/file "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /usr/bin/checksec --file=/mnt/d/题目路径/binary
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools pwn checksec "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools pwn cyclic 200
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools ROPgadget --binary "/mnt/d/题目路径/binary" --only "pop|ret"
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools python "/mnt/d/题目路径/exploit.py"
```

最后一条可能运行本地目标或连接服务，按脚本实际行为和题目授权执行。cyclic 的字母宽度、目标字长和字节序要与偏移恢复一致。

Windows 的 ROPgadget 入口是上述无扩展名脚本，由 venv Python 执行；没有同名 .exe，不能根据发行包名猜测可执行路径。

## GDB、pwndbg 与原生工具

| 工具 | 位置 | 当前版本或来源 | 作用 |
|---|---|---|---|
| GDB / GDB multiarch | `/usr/bin/gdb`、`/usr/bin/gdb-multiarch` | APT `17.2-1+b1` | Linux 调试与跨架构远程调试 |
| pwndbg | `/home/kali/pwndbg/gdbinit.py`；独立 `.venv` | 源码部署；GDB 加载可用 | 寄存器、栈、堆、内存映射和保护检查 |
| objdump / readelf / nm / strings | `/usr/bin/` 下同名入口 | APT `binutils 2.47-2` | 汇编、段节、动态链接和符号信息 |
| checksec | `/usr/bin/checksec` | APT `2.6.0-2` | PIE、RELRO、NX、Canary 等保护 |
| strace / ltrace | `/usr/bin/strace`、`/usr/bin/ltrace` | APT `7.0+ds-1` / `0.7.91~git20230705.8eabf68-4+b1` | 实际运行目标，观测系统调用或库调用 |
| one_gadget | `/usr/local/bin/one_gadget` | Ruby gem `1.10.0` | 查找 libc gadget 及寄存器/栈约束 |
| seccomp-tools | `/usr/local/bin/seccomp-tools` | Ruby gem `1.6.2` | 运行目标并转储 seccomp BPF |
| libc6-dbg | APT 调试符号 | `2.43-4` | 本机 glibc 调试；不代替题目提供的 libc 与符号 |

pwndbg 有独立依赖环境，不属于 `ctf-tools`。调用时禁用自动更新，避免调试器启动时变更环境：

```pwsh
wsl -d kali-linux --exec env PWNDBG_NO_AUTOUPDATE=1 /usr/bin/gdb -q "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /usr/bin/readelf -h -l -d "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /usr/bin/objdump -d -M intel "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /usr/local/bin/one_gadget "/mnt/d/题目路径/libc.so.6"
wsl -d kali-linux --exec /usr/local/bin/seccomp-tools dump "/mnt/d/题目路径/binary"
```

GDB 中按具体目标使用 `start`、`checksec`、`vmmap`、`bt`、`x/16gx $rsp`。没有自动加载 pwndbg 时核对上述源码入口与独立 venv，再在 GDB 中 `source /home/kali/pwndbg/gdbinit.py`；不要自动运行 setup 或安装依赖。陌生 ELF 的依赖首检用 `readelf -d`，实际运行前核对 loader、libc、架构和授权。

## QEMU

| 入口 | APT 包与版本 | 使用边界 |
|---|---|---|
| `/usr/bin/qemu-aarch64`、`qemu-arm`、`qemu-mips`、`qemu-riscv64`、`qemu-i386`、`qemu-x86_64` | `qemu-user 1:11.1.0+ds-2` | Linux 用户态模拟，必要时使用题目匹配的 sysroot |
| `/usr/bin/qemu-system-x86_64` | `qemu-system-x86 1:11.1.0+ds-2` | 内核、initramfs 与完整系统运行 |

用户态示例在运行范围明确后执行；`/path/to/target-sysroot` 是需要提供的目标根文件系统，不是已安装目录：

```pwsh
wsl -d kali-linux --exec /usr/bin/qemu-arm -L /path/to/target-sysroot "/mnt/d/题目路径/arm_binary"
```

内核题沿题目启动脚本核对 CPU、内存、保护位、设备与调试端口，不套通用模板假定能复现。长时间 QEMU、rootfs 编译和 fuzz 先明确预算。

## 失败处理与转向

- 保护信息与 crash 还不足以确定漏洞族：最小化输入，转 [pwn-first-pass-red-flags-and-protections.md](pwn-first-pass-red-flags-and-protections.md)。
- 本地成功、远端失败：对照 libc、PIE、环境变量、I/O 同步、字节序和超时；保护组合见 [runtime-protection-and-tls-exploits.md](runtime-protection-and-tls-exploits.md)。
- gadget 能找到却不能用：检查每条约束和 primitive，必要时转栈迁移、SROP、FSOP 或堆路线；看 [stack-pivots-srop-and-seccomp-rop.md](stack-pivots-srop-and-seccomp-rop.md)。
- QEMU/内核行为不符：先核对题目镜像与启动条件，再查 [linux-kernel-exploit-basics.md](linux-kernel-exploit-basics.md)、[kaslr-kpti-smep-and-kernel-debugging.md](kaslr-kpti-smep-and-kernel-debugging.md)。
- 缺工具不自动安装。WSL 新增、升级、重装须先说明为何依赖 Linux、方式、目标、主要依赖与影响并取得明确同意。
