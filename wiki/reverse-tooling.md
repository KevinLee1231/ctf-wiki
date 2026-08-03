---
type: tooling
tags: [reverse, tooling, tools, environment, ida, idalib, ghidra]
skills: [ctf-reverse, ctf-mobile, ctf-malware, ctf-pwn, ctf-hardware-embedded]
updated: 2026-08-03
---

# Reverse Tooling

本页记录 `ctf-reverse` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 工具选择边界

### 入口选择

- 普通 native binary 首选 IDA Pro MCP（`idalib`）：先发现或打开会话，再用一次全局 survey 建立函数、字符串、导入和调用图画像。
- Ghidra MCP 降为普通静态分析工具；IDA 不可用、处理器/文件格式支持不合适、需要显式 raw language 导入或已有 Ghidra 项目时再切换。
- 首轮仍先用 `file`、`strings`、`capa`、`objdump/readelf` 定性，不要直接进入重型符号执行。
- Python bytecode、Android、firmware、跨架构和 Windows/.NET 各有专用入口，按本页路径调用。

### 不应进入 Reverse 工具链的情况

- 已经确认漏洞 primitive、远程交互和内存破坏路线时，转 Pwn，不把 exploit 阶段写成 RE。
- 证据主要是磁盘/PCAP/metadata/媒体恢复时，先转 Forensics。
- Web、协议或认证链是主障碍时，先转 Web/Pentest，Reverse 只处理其中的本地组件。

### 补工具经验的触发条件

- raw 暴露 IDA Pro MCP 的会话发现、数据库打开、survey、反编译、类型恢复或保存流程。
- raw 暴露 Ghidra MCP 的实例连接、显式 language 导入或既有项目协作流程。
- Mobile/game 题需要 IL2CPP metadata、Unity assets、Android native/Java 交叉工具链。
- VM/obfuscation 题需要 handler table、dispatcher、trace slicing 或 IR lifting 的可复用步骤。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| IDA Pro MCP（`idalib`） | 普通 native binary 的首选分析入口；一次 survey 即可得到全局画像 |
| `file` / `strings` | 先确认载体类型和明显线索 |
| `capa` | 未知 binary 先看能力画像 |
| Ghidra MCP | IDA 不适用时的普通静态分析备选 |
| `/home/kali/pycdc/build/pycdc` | 一旦确认是 `.pyc`，它就是最快路径；当前不在 `PATH` 中 |

### 专项按需

- 静态/动态辅助：`radare2`、GDB+pwndbg、`frida`
- 自动分析与模拟：`angr`、`z3-solver`、`qiling`、`unicorn`
- 文件格式与补丁：`lief`、`pyelftools`
- 打包/平台专项：`pyinstxtractor-ng`、`decompyle3`、`uncompyle6`、`apktool`、`baksmali`、`binwalk`、`unsquashfs`、`upx`
- 跨架构：`qemu-*`

### 当前未装 / 建议按需补装

当前 native、动态调试、符号执行、跨架构、Python bytecode、固件提取，以及 APK 高层反编译和低层解包的 baseline 已经完整，但仍有以下窄缺口：

| 候选 | 当前状态 | 是否值得补 | 边界 |
|---|---|---|---|
| `r2pipe` | `ctf-tools` 未安装，但 radare2 CLI 已有 | 中 | 只在需要批量 JSON/函数级自动化时补；单题手工分析不缺。 |
| Miasm / d810-ng | Miasm 未在 `ctf-tools`；d810-ng 未确认安装 | 中，按题 | 专用于 IR/符号反混淆与 IDA microcode 规则；安装前需核对 IDA/Python 兼容性。 |
| APKiD / Androguard | `ctf-tools` 未安装；WSL `uv` 0.11.8 可用 | 中，按 APK 频率 | 两者都是 Python 项目；APKiD 以 CLI 为主，适合 `uv tool`，Androguard 若需脚本导入则使用独立 `uv venv`。不要写入现有 `ctf-tools`。 |
| `bbpb` (Blackbox Protobuf) | `ctf-tools` 未安装；旧包名 `blackboxprotobuf` 也未安装 | 中低，协议题按需 | 当前上游把 `bbpb` 列为官方 PyPI 包；只在缺少 `.proto` 且 protobuf 编码是主障碍时补。 |
| jshookmcp | 当前没有可调用的对应 MCP | 中低，浏览器 JS 频繁时 | 会引入新服务/配置层；本次只列候选，不注册。 |
| EMBA/Firmadyne 类固件框架 | 常用 WSL 路径未发现 | 低，仅固件题频繁时 | 体积、依赖和运行成本高；现有 `binwalk`/`unsquashfs`/QEMU/Qiling 先用。 |

`anything-analyzer`、公开去混淆服务、自动 bootstrap 和自建 IDA HTTP server 不列为本机 baseline：前两者与现有工具重叠或有样本外传边界，后两者会改变环境/会话治理。

#### APKiD / Androguard 的 uv 安装边界

- APKiD 3.1.0 是 Python CLI，入口为 `apkid`，用于识别 compiler、packer、protector、obfuscator 和其他 APK/DEX 特征；其运行依赖是 `yara-python-dex>=1.0.1`。
- Androguard 4.1.4 是 Python 3.9+ 的库和 CLI，入口为 `androguard`，覆盖 APK/DEX、binary XML、resources、反汇编和可编程分析。4.x 与旧版 3.3.5 API 有明显差异，旧脚本迁移时不能只替换版本号。
- 不把两者直接装入 `ctf-tools`：APKiD 的 `yara-python-dex` 与环境中已有的 `yara-python` 都提供 `yara` 模块；Androguard 的依赖更新也不应影响通用 CTF 环境。
- APKiD 使用命令级 `UV_TOOL_DIR=/home/kali/ctf-uv-tools` 和 `UV_TOOL_BIN_DIR=/home/kali/ctf-uv-tools/bin` 创建持久隔离工具环境，不执行 `uv tool update-shell`，后续调用 `/home/kali/ctf-uv-tools/bin/apkid`。
- Androguard 仅用 CLI 时也可安装到该 `uv tool` 层；若 solver/分析脚本需要 `import androguard`，则创建 `/home/kali/androguard-venv`，并固定使用其中的 `bin/python` 和 `bin/androguard`，不要假设 `ctf-tools` 可以导入它。

## 失败信号与转向

- IDA 会话、Hex-Rays 或自动分析不可用：先保留 `file`/字符串/导入/汇编事实；普通格式可转 Ghidra MCP，运行时生成代码则直接转动态 dump，不要只反复更换反编译器。
- IDA/Ghidra 伪代码都不可读：回到汇编、字符串、交叉引用和运行时断点；若是 VM/壳/SMC，转 [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md) 或 [packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)。
- 程序一运行就退出、检测调试器或时间环境：先转 [anti-analysis.md](anti-analysis.md)，不要继续换反编译器。
- 只差最终输入但比较点可断：转 [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)，优先抓明文 buffer 或最终比较参数。
- 目标依赖完整系统调用、rootfs、跨架构环境或固件：转 [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md) 或 [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)。

## 详细清单

RE 优先使用 **IDA Pro MCP（`idalib`）**。

### IDA Pro MCP（首选）

当前 MCP 工具已可调用；2026-08-03 审计时 `idb_list` 返回 0 个会话。这表示“当时无已打开数据库”，不表示 IDA MCP 缺失。

稳定调用顺序：

1. `idb_list`：发现已经被 MCP 接管的 GUI/worker 会话；不要猜测 database/session id。
2. `idb_open`：没有目标会话时按绝对路径打开 binary。自动化默认可用 headless；需要沿用当前 GUI 数据库时选择 GUI 模式，并按需启用自动分析、Hex-Rays 和缓存。
3. `survey_binary`：作为打开后的首个分析调用；普通目标用 `standard`，函数超过约一万时用 `minimal`，不要先分别拉全量函数、导入和字符串。
4. 根据 survey 证据再用 `decompile`、`analyze_component`、交叉引用、重命名和类型恢复工具；不要无目标地批量反编译。
5. 只有确实修改数据库且需要持久化时才调用 `idb_save`；GUI 会话保存到当前数据库，headless 会话才适合打包为独立 `.i64/.idb`。

| 状态 | 处理 |
|---|---|
| `idb_list` 没有目标会话 | 用 `idb_open` 打开绝对路径，不把“无 GUI”误判为 MCP 不可用。 |
| `idb_open` 或 Hex-Rays 初始化失败 | 保留静态首检结果；尝试不依赖伪代码的汇编/交叉引用，或转 Ghidra MCP。 |
| raw firmware 架构无法可靠识别 | 先确认架构、端序和装载基址；需要显式 language 导入时可转 Ghidra。 |
| 大型 binary survey 输出过重 | 改用 `detail_level="minimal"`，再围绕少量候选函数深挖。 |

### Ghidra MCP（普通工具）

当前 MCP 工具已可调用；2026-08-03 审计时 `list_instances` 没有正在运行的实例。这是运行状态，不是安装结论。

- 先用 `list_instances` 检查实例；需要项目级分析时再 `connect_instance`。
- 连接成功后工具列表会动态扩展，随后才调用反编译、交叉引用、导入或调试能力。
- 没有实例时，不把静态桥接工具误认为已经进入项目；需要导入 raw firmware 时显式给出 language/编译器规格。

| 适用情况 | 调用方式 |
|---|---|
| IDA 不可用、处理器支持不合适或已有 Ghidra 项目 | `list_instances` → `connect_instance` → 项目分析工具 |
| raw firmware 需要显式 language 导入 | 连接项目后使用 `import_file`，明确 `language`，再等待分析完成 |

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
| **GDB+pwndbg** | `/usr/bin/gdb` 17.2 + pwndbg | 动态调试 | `gdb ./bin; start; b *main` |
| **objdump** | `/usr/bin/objdump` | 反汇编 | `objdump -M intel -d bin` |
| **strings** | `/usr/bin/strings` | 提取可打印字符串 | `strings bin \| grep -i flag` |
| **file** | `/usr/bin/file` | 识别文件类型/架构 | `file binary` |
| **readelf** | `/usr/bin/readelf` | ELF 结构分析 | `readelf -S binary; readelf -l binary` |
| **nm** | `/usr/bin/nm` | 符号表列出 | `nm binary \| grep main` |
| **apktool** | `/usr/bin/apktool` 2.7.0 | APK 解包 | `apktool d app.apk -o decoded/` |
| **strace** | `/usr/bin/strace` 7.0 | 系统调用跟踪 | `strace -f ./bin` |
| **ltrace** | `/usr/bin/ltrace` 0.7.91 | 库函数调用跟踪 | `ltrace ./bin` |
| **upx** | `/usr/bin/upx` 4.2.4 | 脱壳 | `upx -d packed -o unpacked` |
| **one_gadget** | `/usr/local/bin/one_gadget` 1.10.0 | libc gadget 查找 | `one_gadget libc.so.6` |
| **binwalk** | `/usr/bin/binwalk` 2.4.3 | 固件/嵌入文件提取 | `binwalk -Me firmware.bin` |
| **seccomp-tools** | `/usr/local/bin/seccomp-tools` 1.6.2 | SECCOMP/BPF 转储 | `seccomp-tools dump ./bin` |
| **baksmali** | `/usr/bin/baksmali` 2.5.2 | DEX 字节码反汇编 | `baksmali d classes.dex -o smali/` |
| **unsquashfs** | `/usr/bin/unsquashfs` 4.7.5 | SquashFS 提取 | `unsquashfs -d out/ firmware.sqfs` |
| **qemu-riscv64** | `/usr/bin/qemu-riscv64` 11.0.1 | RISC-V 模拟 | `qemu-riscv64 -L /usr/riscv64-linux-gnu/ ./bin` |
| **qemu-arm** | `/usr/bin/qemu-arm` 11.0.1 | ARM 模拟 | `qemu-arm -L /usr/arm-linux-gnueabihf/ ./bin` |
| **qemu-mips** | `/usr/bin/qemu-mips` 11.0.1 | MIPS 模拟 | `qemu-mips -L /usr/mips-linux-gnu/ ./bin` |

### Windows 本地

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **dnSpy** | `D:\CTF工具\dnSpy-net-win64\dnSpy.exe` | .NET 反编译/调试 | GUI 打开 binary 或 Assembly-CSharp.dll |
| **JADX 1.5.6** | `D:/CTF工具/jadx-1.5.6/bin/jadx.bat`；GUI 为同目录 `jadx-gui.bat` | APK/DEX/AAB/APKM 的 Java-like 反编译、资源查看、搜索、xref、单类恢复和调用图导出 | `& "D:/CTF工具/jadx-1.5.6/bin/jadx.bat" -d "jadx-output" "app.apk"`；GUI：`& "D:/CTF工具/jadx-1.5.6/bin/jadx-gui.bat" "app.apk"` |
| **Node.js** | `"C:/Program Files/nodejs/node.exe"` 24.13.0 | 浏览器 JavaScript 的最小离线 harness | 只引入已捕获的必要环境值，按首个状态分歧点补 mock |
| **OpenJDK** | `"C:/Program Files/Eclipse Adoptium/jdk-21.0.10.7-hotspot/bin/java.exe"` 21.0.10 | APK/JAR 工具的 Java 运行时 | `JAVA_HOME` 已指向该 JDK，并已验证可运行 JADX 1.5.6 |
| **Manifest summary** | `C:/Users/LMY/.agents/skills/ctf-reverse/scripts/manifest-summary.ps1` | 只读摘要已解码 Android Manifest 的 SDK、权限、组件、intent-filter 和 launcher | `& "C:/Users/LMY/.agents/skills/ctf-reverse/scripts/manifest-summary.ps1" -ManifestPath "decoded/AndroidManifest.xml"` |

#### JADX 1.5.6 固定用法

```powershell
# 版本与帮助
& "D:/CTF工具/jadx-1.5.6/bin/jadx.bat" --version
& "D:/CTF工具/jadx-1.5.6/bin/jadx.bat" --help

# GUI 导航
& "D:/CTF工具/jadx-1.5.6/bin/jadx-gui.bat" "D:/题目路径/app.apk"

# CLI 全量输出；输出目录应位于当前题目工作区
& "D:/CTF工具/jadx-1.5.6/bin/jadx.bat" -d "D:/题目路径/jadx-output" "D:/题目路径/app.apk"

# 只恢复源码，跳过 resources
& "D:/CTF工具/jadx-1.5.6/bin/jadx.bat" --no-res -d "D:/题目路径/jadx-sources" "D:/题目路径/app.apk"
```

- 默认先用 `auto` 模式导航；结构化伪代码明显失真时，用 `--decompilation-mode simple` 或 `fallback` 对照，不把 `--show-bad-code` 产生的代码当作正确语义。
- JADX 用于 Java/Kotlin-like 视图、搜索和 xref；关键条件、异常处理、JNI 边界和反编译失败函数仍用 `baksmali`/DEX 或 native 工具核验。
- 保持 `JADX_DISABLE_XML_SECURITY`、`JADX_DISABLE_ZIP_SECURITY` 未设置；不为处理异常附件而关闭 ZIP/XML 安全限制。

### 专项全路径工具

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| **pycdc** | `/home/kali/pycdc/build/pycdc` | Python 3.9+ `.pyc` 反编译主力；旧版本字节码按需与 `decompyle3` / `uncompyle6` 交叉验证 | `/home/kali/pycdc/build/pycdc file.pyc` |
| **pycdas** | `/home/kali/pycdc/build/pycdas` | Python 字节码反汇编 | `/home/kali/pycdc/build/pycdas file.pyc` |
| **capa** | `/home/kali/miniforge3/envs/ctf-tools/bin/capa` | `ctf-tools` 内命令行工具；自动识别 binary 能力（加密/通信/反分析） | `conda activate ctf-tools && capa -vv binary && conda deactivate` |
| **decompyle3** | `/home/kali/.local/bin/decompyle3` | 用户层 Python bytecode 反编译器；只在传统 Python 反编译链更适合时，作为 `pycdc` 的补充交叉验证 | `/home/kali/.local/bin/decompyle3 file.pyc` |
