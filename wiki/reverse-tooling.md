---
type: tooling
tags: [reverse, tooling, tools, environment]
skills: [ctf-reverse]
updated: 2026-05-21
---

# Reverse Tooling

本页记录 `ctf-reverse` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 调用层与覆盖状态

### 非交互调用原则

- 普通 binary 首选 Ghidra MCP；Ghidra 未启动时只能得到有限桥接能力，启动并打开项目后再使用完整分析工具。
- 首轮仍先用 `file`、`strings`、`capa`、`objdump/readelf` 定性，不要直接进入重型符号执行。
- Python bytecode、Android、firmware、跨架构和 Windows/.NET 各有专用入口，按本页路径调用。

### 知识页覆盖状态

- 当前覆盖普通逆向、动态调试、VM/壳、Python/bytecode、Go/Rust/JVM、硬件/firmware、Windows kernel 和多道近期 WP。
- 后续缺口主要是 Mach-O/iOS、Swift/Objective-C、Unity/IL2CPP、更多 VM handler lifting 和自动化 trace 模板。

### 后续补强方向

- Ghidra MCP workflow：实例连接、函数命名、手动 create function、trace 证据保存。
- Mobile/game：IL2CPP metadata、Unity assets、Android native/Java 交叉。
- VM/obfuscation：handler table、dispatcher、trace slicing、IR lifting。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| Ghidra MCP | 普通二进制的主分析入口 |
| `file` / `strings` | 先确认载体类型和明显线索 |
| `capa` | 未知 binary 先看能力画像 |
| `pycdc` | 一旦确认是 `.pyc`，它就是最快路径 |

### 专项按需

- 静态/动态辅助：`radare2`、GDB+pwndbg、`frida`
- 自动分析与模拟：`angr`、`z3-solver`、`qiling`、`unicorn`
- 文件格式与补丁：`lief`、`pyelftools`
- 打包/平台专项：`pyinstxtractor-ng`、`decompyle3`、`uncompyle6`、`apktool`、`baksmali`、`binwalk`、`unsquashfs`、`upx`
- 跨架构：`qemu-*`

### 当前未装 / 建议按需补装

- 当前没有明显基础缺口。RE 这里真正更重要的是保持 Ghidra MCP 的连接流程和项目能力说明准确。

## 详细清单

RE 优先使用 **Ghidra MCP**。

- Ghidra 未启动时，桥接层通常只暴露少量静态工具；先检查是否存在可连接实例。
- 启动 Ghidra 并打开项目后，再连接实例进入完整分析模式，随后才使用反编译、交叉引用、调试与 trace 等项目级能力。

### Ghidra MCP（首选）

| 工具 | 功能 | 调用方式 |
|---|---|---|
| Ghidra MCP | 反编译/函数分析/交叉引用/调试/trace | 先用 `list_instances` 检查实例；Ghidra 已启动时再 `connect_instance`，连接后使用完整分析工具 |

### Python 包（ctf-tools conda）

| 工具 | 版本 | 功能 | 典型用法 |
|---|---|---|---|
| **angr** | 9.2.209 | 符号执行/CFG 恢复/路径探索 | `angr.Project("./bin").simgr.explore(find=addr)` |
| **z3-solver** | 4.13.0 | SMT 约束求解（序列号/分支条件） | `Solver().add(...); s.check(); s.model()` |
| **capstone** | 5.0.6 | 多架构反汇编引擎 | `Cs(CS_ARCH_X86, CS_MODE_64).disasm(code, base)` |
| **unicorn** | 2.1.2 | CPU 模拟器（x86/ARM/MIPS/SPARC） | `Uc(UC_ARCH_X86, UC_MODE_64).emu_start(b,e)` |
| **pwntools** | 4.15.0 | ELF 补丁/checksec/GOT/PLT | `ELF("./bin"); elf.asm(addr, "ret"); elf.save()` |
| **frida** | 17.9.1 | 动态插桩核心库 | `frida -f ./bin -l hook.js` |
| **frida-tools** | 14.8.1 | Frida CLI（frida-ps/frida-trace） | `frida-trace -i 'strcmp' ./bin` |
| **qiling** | 1.4.6 | 跨平台模拟（Linux/Win/ARM/MIPS/UEFI） | `Qiling(["./bin"], "rootfs/x8664_linux").run()` |
| **lief** | 0.17.6 | ELF/PE/Mach-O 解析与修改 | `lief.parse("bin").patch_pltgot("strcmp", addr)` |
| **keystone-engine** | 0.9.2 | 多架构汇编引擎 | `Ks(KS_ARCH_X86, KS_MODE_64).asm("ret")` |
| **pyelftools** | 0.32 | ELF 文件解析 | `ELFFile(f).get_section_by_name(".rodata")` |

### 系统全局命令（WSL Kali）

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **radare2** | `/usr/bin/r2` 6.0.5 | 命令行逆向框架 | `r2 -d ./bin; aaa; afl; pdf @ main` |
| **GDB+pwndbg** | `/usr/bin/gdb` 17.1 + pwndbg | 动态调试 | `gdb ./bin; start; b *main` |
| **objdump** | `/usr/bin/objdump` | 反汇编 | `objdump -M intel -d bin` |
| **strings** | `/usr/bin/strings` | 提取可打印字符串 | `strings bin \| grep -i flag` |
| **file** | `/usr/bin/file` | 识别文件类型/架构 | `file binary` |
| **readelf** | `/usr/bin/readelf` | ELF 结构分析 | `readelf -S binary; readelf -l binary` |
| **nm** | `/usr/bin/nm` | 符号表列出 | `nm binary \| grep main` |
| **apktool** | `/usr/bin/apktool` 2.7.0 | APK 解包 | `apktool d app.apk -o decoded/` |
| **strace** | `/usr/bin/strace` 6.18 | 系统调用跟踪 | `strace -f ./bin` |
| **ltrace** | `/usr/bin/ltrace` 0.7.91 | 库函数调用跟踪 | `ltrace ./bin` |
| **upx** | `/usr/bin/upx` 4.2.4 | 脱壳 | `upx -d packed -o unpacked` |
| **one_gadget** | `/usr/local/bin/one_gadget` 1.10.0 | libc gadget 查找 | `one_gadget libc.so.6` |
| **binwalk** | `/usr/bin/binwalk` 2.4.3 | 固件/嵌入文件提取 | `binwalk -Me firmware.bin` |
| **seccomp-tools** | `/usr/local/bin/seccomp-tools` 1.6.2 | SECCOMP/BPF 转储 | `seccomp-tools dump ./bin` |
| **baksmali** | `/usr/bin/baksmali` 2.5.2 | DEX 字节码反汇编 | `baksmali d classes.dex -o smali/` |
| **unsquashfs** | `/usr/bin/unsquashfs` 4.7.5 | SquashFS 提取 | `unsquashfs -d out/ firmware.sqfs` |
| **qemu-riscv64** | `/usr/bin/qemu-riscv64` 10.2.2 | RISC-V 模拟 | `qemu-riscv64 -L /usr/riscv64-linux-gnu/ ./bin` |
| **qemu-arm** | `/usr/bin/qemu-arm` 10.2.2 | ARM 模拟 | `qemu-arm -L /usr/arm-linux-gnueabihf/ ./bin` |
| **qemu-mips** | `/usr/bin/qemu-mips` 10.2.2 | MIPS 模拟 | `qemu-mips -L /usr/mips-linux-gnu/ ./bin` |

### Windows 本地

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **dnSpy** | `D:\CTF工具\dnSpy-net-win64\dnSpy.exe` | .NET 反编译/调试 | GUI 打开 binary 或 Assembly-CSharp.dll |

### 专项全路径工具

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **pycdc** | `/home/kali/pycdc/build/pycdc` | Python 3.9+ `.pyc` 反编译主力；旧版本字节码按需与 `decompyle3` / `uncompyle6` 交叉验证 | `/home/kali/pycdc/build/pycdc file.pyc` |
| **pycdas** | `/home/kali/pycdc/build/pycdas` | Python 字节码反汇编 | `/home/kali/pycdc/build/pycdas file.pyc` |
| **capa** | `/home/kali/miniforge3/envs/ctf-tools/bin/capa` | `ctf-tools` 内命令行工具；自动识别 binary 能力（加密/通信/反分析） | `conda activate ctf-tools && capa -vv binary && conda deactivate` |
| **decompyle3** | `/home/kali/.local/bin/decompyle3` | 用户层 Python bytecode 反编译器；只在传统 Python 反编译链更适合时，作为 `pycdc` 的补充交叉验证 | `/home/kali/.local/bin/decompyle3 file.pyc` |
