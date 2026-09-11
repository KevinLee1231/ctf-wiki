---
type: tooling
tags: [reverse, tooling, tools, environment, ida, idalib, idapython, ghidra, x64dbg]
skills: [ctf-reverse]
---

# Reverse Tooling

本页维护 Reverse 方向本机工具的状态、路径、调用与失败处理。相关方向直接引用本页的静态分析、IDAPython、JADX 和 Windows 调试入口，不复制另一份工具目录。

## 平台与入口选择

普通静态分析、ELF/字节码解析、约束和局部 CPU 模拟优先在 Windows 完成；目标是 Linux 程序本身不构成使用 WSL 的理由。只有实际运行 Linux 目标、ptrace/GDB、Linux 本地 Frida 或跨架构用户态环境需要 WSL。首轮先确认载体、架构与少量字符串，再围绕具体函数深入。

Windows Python 为 `D:/文档/新建文件夹/venv/Scripts/python.exe`（3.14.3）。当前可用分析包为：

| Windows venv 包 | 版本 | 用途 |
|---|---|---|
| `capstone` | `5.0.7` | 多架构反汇编 |
| `unicorn` | `2.1.4` | 有界 CPU 指令模拟，不提供完整操作系统语义 |
| `pyelftools` | `0.32` | ELF header、section、segment 和符号读取 |
| `z3-solver` | `4.16.0.0` | 已还原逻辑的约束求解 |
| `ROPGadget` | `7.7` | gadget 搜索；利用阶段转 [pwn-tooling.md](pwn-tooling.md) |

```pwsh
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/analyze.py"
wsl -d kali-linux --exec /usr/bin/file "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /usr/bin/readelf -h -S "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /usr/bin/strings -n 8 "/mnt/d/题目路径/binary"
```

`D:/题目路径` 与 WSL `/mnt/d/题目路径` 指同一 Windows 目录。APT 命令直接从 `pwsh` 调 WSL，不套 Conda；MCP 是会话工具，不是 shell 命令。

当前 Windows venv 与 WSL `ctf-tools` 均未发现 angr、Qiling、LIEF、capa；旧用户层反编译器和 APKiD/Androguard 隔离入口也已不存在。不要运行旧清单中的命令或把“没有装包”当作必须转 WSL 的证据。需要额外工具时先评估 Windows 兼容版本及当前分析方法是否足够。

## Native 静态分析与 IDAPython

普通 native binary 首选 IDA Pro MCP：发现/打开目标后先做一次全局 survey。IDA 不可用、处理器支持不合适、需要显式 raw language 或已有 Ghidra 项目时，使用 Ghidra。

### IDA Pro MCP（首选）

当前 IDA MCP 的会话查询接口可调用；打开目标前重新读取会话列表。没有会话不表示 MCP 缺失，安装目录名也不能作为实际运行版本的证明。

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

### IDAPython 脚本开发

需要自定义批量提取、类型构造、ctree 遍历或 IDA 插件代码时，读取 [idapython skill](C:/Users/LMY/.agents/skills/idapython/SKILL.md) 并按模块查询其参考资料。直接调用现成的反编译、交叉引用或重命名 MCP 工具时，不需要额外加载脚本开发资料。

执行脚本必须依附已初始化的 IDA/IDAPython 运行时；普通项目 Python 或 MCP 客户端不能直接替代它。先确认 GUI/headless/包装器的实际执行方式和可调用接口，再运行本次脚本。本页不假定某个 MCP 服务必然提供任意 Python 执行功能；只有查询接口时沿现有接口完成查询，脚本需转到明确可用的 IDA 执行入口。

### Ghidra MCP（普通工具）

当前 Ghidra MCP 的实例查询接口可调用；本地目录 `D:/CTF工具/ghidra_12.1.2_PUBLIC` 的 application.properties 标明版本 12.1.2。实例是否已启动、连接了哪个项目须使用前查询，文件存在不表示项目已就绪。

- 先用 `list_instances` 检查实例；需要项目级分析时再 `connect_instance`。
- 连接成功后工具列表会动态扩展，随后才调用反编译、交叉引用、导入或调试能力。
- 没有实例时，不把静态桥接工具误认为已经进入项目；需要导入 raw firmware 时显式给出 language/编译器规格。

| 适用情况 | 调用方式 |
|---|---|
| IDA 不可用、处理器支持不合适或已有 Ghidra 项目 | `list_instances` → `connect_instance` → 项目分析工具 |
| raw firmware 需要显式 language 导入 | 连接项目后使用 `import_file`，明确 `language`，再等待分析完成 |

## WSL 原生运行与系统工具

`ctf-tools` 保留 `pwntools 4.15.0`、`frida 17.9.1`、`frida-tools 14.8.1` 及必要依赖；其 Python 3.11.15 位于 `/home/kali/miniforge3/envs/ctf-tools`。Linux 本地 Frida CLI 可用，其他平台、USB 设备或 frida-server 连接必须另行核对。通用 Python 分析仍使用 Windows。

```pwsh
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools frida --version
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools frida-ps
wsl -d kali-linux --exec /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools frida -f "/mnt/d/题目路径/binary" -l "/mnt/d/题目路径/hook.js"
```

最后一条实际运行目标，只在本次运行授权范围内使用。普通 APK 静态分析或连接远程设备不因已有这套 Frida 就默认走 WSL。

| WSL 工具 | 路径 | APT 包版本或部署状态 | 用途 |
|---|---|---|---|
| radare2 | `/usr/bin/r2` | `6.0.4+dfsg-1` | CLI 静态分析、汇编和项目检查 |
| objdump / readelf / nm / strings | `/usr/bin/` 下同名入口 | `binutils 2.47-2` | ELF 与汇编结构 |
| GDB + pwndbg | `/usr/bin/gdb`、`/home/kali/pwndbg/gdbinit.py` | GDB `17.2-1+b1`，pwndbg 独立 venv | Linux 动态调试，完整约定见 [pwn-tooling.md](pwn-tooling.md) |
| strace / ltrace | `/usr/bin/strace`、`/usr/bin/ltrace` | `7.0+ds-1` / `0.7.91~git20230705.8eabf68-4+b1` | 系统调用与库调用 trace |
| upx | `/usr/bin/upx` | `upx-ucl 4.2.4-1.1` | 支持格式的 UPX 解包 |
| binwalk / unsquashfs | `/usr/bin/binwalk`、`/usr/bin/unsquashfs` | `2.4.3+dfsg1-3` / `1:4.7.5-1` | 固件嵌入识别与 SquashFS 提取 |
| wasm2wat / wat2wasm | `/usr/bin/wasm2wat`、`/usr/bin/wat2wasm` | `wabt 1.0.41+dfsg+~cs1.0.39-2` | WASM 二进制与文本转换 |
| pycdc / pycdas | `/home/kali/pycdc/build/pycdc`、`/home/kali/pycdc/build/pycdas` | 手工构建，入口与动态库存在 | Python 字节码反编译/反汇编；不保证支持所有新版本 opcode |

```pwsh
wsl -d kali-linux --exec env PWNDBG_NO_AUTOUPDATE=1 /usr/bin/gdb -q "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /usr/bin/r2 -q -c "iI;is" "/mnt/d/题目路径/binary"
wsl -d kali-linux --exec /home/kali/pycdc/build/pycdc "/mnt/d/题目路径/module.pyc"
wsl -d kali-linux --exec /home/kali/pycdc/build/pycdas "/mnt/d/题目路径/module.pyc"
wsl -d kali-linux --exec /usr/bin/wasm2wat "/mnt/d/题目路径/module.wasm" -o "/mnt/d/题目路径/module.wat"
wsl -d kali-linux --exec /usr/bin/binwalk "/mnt/d/题目路径/firmware.bin"
```

pwndbg 启动时固定 `PWNDBG_NO_AUTOUPDATE=1`，避免自动变更环境。跨架构 QEMU 与目标 sysroot 的调用见 [pwn-tooling.md](pwn-tooling.md)，硬件外设与启动链见 [hardware-embedded-tooling.md](hardware-embedded-tooling.md)。

## Windows 调试与字节码工具

#### x64dbg / x32dbg

适用于 Windows PE 的条件断点、参数观测、线程/异常分析和内存 patch 验证。分析方法见 [frida-angr-lldb-and-x64dbg.md](frida-angr-lldb-and-x64dbg.md)；本节维护本机入口及调用。

| 层次 | 当前事实与调用边界 |
|---|---|
| 64 位调试器 | 已发现 `D:/CTF工具/x64dbg/x64/x64dbg.exe`；PE 文件版本字段为 `0.0.2.5`，该字段不代表已核实具体快照构建日期 |
| 32 位调试器 | 已发现 `D:/CTF工具/x64dbg/x32/x32dbg.exe`；PE 文件版本字段为 `0.0.2.5` |
| 本地 MCP 插件 | 已发现 `D:/CTF工具/x64dbg/x64/plugins/x64dbg_mcp.dp64`；文件未提供版本字段，文件存在不等于插件已加载或 MCP 已连接 |
| 当前任务 MCP | 当前会话没有暴露 x64dbg MCP 工具；不能按离线工具目录宣称接口可调用，也不能据此判定调试器未安装 |

GUI 入口按目标 PE 位数选择一条，目标路径替换为本次样本；启动样本属于实际运行操作：

```pwsh
# 64 位 PE
& "D:/CTF工具/x64dbg/x64/x64dbg.exe" "D:/题目路径/sample.exe"

# 32 位 PE
& "D:/CTF工具/x64dbg/x32/x32dbg.exe" "D:/题目路径/sample.exe"
```

需要 MCP 自动化时先重新发现工具，核对所属后端、目标进程/会话、当前状态与参数 schema。只有当前确实暴露的接口才可调用；不按旧目录拼接工具名、端点或固定工具数量。连接可用后的顺序是：读取调试状态和目标模块 → 确认模块基址、线程及观察地址 → 读取寄存器/内存 → 设置本次需要的断点 → 运行到观察点 → 读取命中状态并验证结果。修改内存或寄存器前记录原值，清理时只处理本次创建的断点和补丁。

##### 条件断点与日志

先用 GUI 设置软件断点并打开 Edit breakpoint，再设置 break condition、log condition 和 log text；MCP 可用时用当前对应接口表达相同设置。以下为 x64dbg 原生日志文本，不是 JSON 参数或 shell 命令。

在标准 Windows x64 整数/指针参数函数的入口、栈尚未调整时：

```text
entry p1={x:RCX} p2={x:RDX} p3={x:R8} p4={x:R9} p5={x:[RSP+28]} ret={a:[RSP]}
```

`28` 按十六进制解释，位于返回地址和 32 byte shadow space 之后；浮点参数、隐藏参数、特殊调用约定及函数中部断点必须按实际 ABI/栈布局另行解释。x86 栈传参函数入口可记录：

```text
entry p1={x:[ESP+4]} p2={x:[ESP+8]} p3={x:[ESP+C]} ret={a:[ESP]}
```

常用格式为 `{x:RCX}`（十六进制）、`{a:[RSP]}`（返回地址信息）、`{utf16@RCX}`（RCX 指向的 UTF-16 字符串）、`{ascii@RCX}`（ASCII 字符串）。字符串指针必须来自实际原型并指向可读内存，不能多解引用一次。

只记录日志而不暂停时，break condition 设为 `0`、log condition 设为 `1`，并关闭 fast resume；否则 break condition 为 0 时会提前跳过日志。条件表达式与日志格式分别填写。表达式中的整数默认十六进制，十进制使用 `.123` 等写法；无效条件可能导致总是命中，应先在少量调用上确认。断点命令中不嵌入 `run` 等改变运行状态的命令。

| 失败状态 | 下一步 |
|---|---|
| MCP 工具未出现或插件连接失败 | 保留已确认的本地 GUI 入口；排查当前插件/连接配置，不猜测接口，也不把静态 IDA 查询当作动态执行结果 |
| 调试器未加载目标或状态为 stopped | 先确认本次目标及运行条件，再通过已验证入口打开；不直接发起单步或读寄存器 |
| 断点未命中 | 检查模块加载时机、模块基址加 RVA、目标位数及线程；模块重载后重新定位 |
| 条件断点总是停住或日志缺失 | 分别核对条件语法、log condition、fast resume、指针有效性与函数入口位置 |
| 调试后输出或控制流变化 | 转 [anti-analysis.md](anti-analysis.md)，核对异常、时间检测及被修改的分支 |

参考：[x64dbg 日志格式](https://help.x64dbg.com/en/latest/introduction/Formatting.html)、[条件断点](https://help.x64dbg.com/en/latest/introduction/ConditionalBreakpoint.html)、[数值语法](https://help.x64dbg.com/en/latest/introduction/Values.html)、[Microsoft x64 调用约定](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention)。

#### 其他 Windows 工具

| 工具 | 路径与状态 | 用途 |
|---|---|---|
| dnSpy | `D:/CTF工具/dnSpy-net-win64/dnSpy.exe`，文件存在 | .NET 反编译、IL 与调试；未启动样本不代表已验证运行行为 |
| JADX | `D:/CTF工具/jadx-1.5.6/lib/jadx-1.5.6-all.jar`，CLI 版本 `1.5.6` | Java/Kotlin-like 视图、APK/DEX 资源与交叉引用 |
| Node.js | `C:/Program Files/nodejs/node.exe`，`24.13.0` | JavaScript 最小离线 harness；浏览器对象按观察结果补 mock |
| OpenJDK | `C:/Program Files/Eclipse Adoptium/jdk-21.0.10.7-hotspot/bin/java.exe`，`21.0.10` | 当前 JADX 可用 Java 运行时 |
| Manifest summary | `C:/Users/LMY/.agents/skills/ctf-reverse/scripts/manifest-summary.ps1` | 只读摘要已解码的 XML Manifest，不能直接读取 APK 内二进制 AXML |

#### JADX 与 Manifest 调用

从 `pwsh` 直接运行 Java jar，不经过 Windows 批处理 shell。先指定本次文件与输出目录：

```pwsh
& "C:/Program Files/Eclipse Adoptium/jdk-21.0.10.7-hotspot/bin/java.exe" -jar "D:/CTF工具/jadx-1.5.6/lib/jadx-1.5.6-all.jar" --version
& "C:/Program Files/Eclipse Adoptium/jdk-21.0.10.7-hotspot/bin/java.exe" -jar "D:/CTF工具/jadx-1.5.6/lib/jadx-1.5.6-all.jar" -d "D:/题目路径/jadx-output" "D:/题目路径/app.apk"
& "C:/Program Files/Eclipse Adoptium/jdk-21.0.10.7-hotspot/bin/java.exe" -jar "D:/CTF工具/jadx-1.5.6/lib/jadx-1.5.6-all.jar" --no-res --single-class "com.example.MainActivity" --single-class-output "D:/题目路径/MainActivity.java" "D:/题目路径/app.apk"
& "C:/Users/LMY/.agents/skills/ctf-reverse/scripts/manifest-summary.ps1" -ManifestPath "D:/题目路径/jadx-output/resources/AndroidManifest.xml"
```

默认 `auto` 模式不能正确恢复控制流时，用 `--decompilation-mode simple` 或 `fallback` 对照；`--show-bad-code` 只显示不完整结果，不证明语义正确。关键分支、异常与 JNI 边界回到 DEX/smali 或 native 汇编。APK 低层解包调用见 [mobile-tooling.md](mobile-tooling.md)。

保持 `JADX_DISABLE_XML_SECURITY`、`JADX_DISABLE_ZIP_SECURITY` 未设置，不为异常附件自动关闭 ZIP/XML 检查。Manifest 摘要路径取决于实际输出布局，先确认文件存在。

## 失败处理与转向

- IDA/Ghidra 会话或伪代码不可用：保留汇编、字符串、导入与交叉引用事实；VM/SMC/壳转 [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)、[packers-deobfuscation-and-debug-automation.md](packers-deobfuscation-and-debug-automation.md)。
- 运行时退出或检测调试器：查 [anti-analysis.md](anti-analysis.md)；比较点可断时优先 [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)。
- pycdc 遇到未知 opcode 或输出错误：核对 magic 与目标 Python 版本，保留 pycdas 指令，不能把“Python 3.9+”当作完整支持保证。确需新反编译器先评估 Windows 兼容版本。
- 局部 Unicorn 模拟缺 syscall、loader 或外设：先明确最小执行边界，再看 [qiling-triton-pin-and-ldpreload.md](qiling-triton-pin-and-ldpreload.md) 或 [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)；方法页提到工具不表示本机已安装。
- 任何 WSL 新增、升级、重装，须先说明用途、Linux 必要性、方式、目标及主要依赖影响并取得明确同意。
