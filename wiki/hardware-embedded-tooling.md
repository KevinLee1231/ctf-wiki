---
type: tooling
tags: [hardware-embedded, tooling, tools, environment]
skills: [ctf-hardware-embedded]
---

# Hardware / Embedded Tooling

本页是 `ctf-hardware-embedded` 方向本机工具信息的唯一权威来源，维护当前安装状态、版本、路径、环境、完整调用、适用边界和失败处理。`ctf-hardware-embedded/SKILL.md` 只说明何时选择某类工具或知识页，不复制本页细节。

本页只描述当前真实状态；实际环境与本文不一致时，直接修正文中现状，不在本页累积核验记录或旧版本历史。

## 工具选择边界

- 固件文件先确认格式、架构、端序、分区和压缩层；逻辑/RF/侧信道输入先保存原始 capture、采样率、位宽、电平和通道定义。
- 真实设备的写 flash、解锁、故障注入、电压改变或擦除操作可能不可恢复；本页命令只覆盖只读首检和离线分析。
- 已提取代码的普通软件语义成为主障碍后转 Reverse；已采集证据的事件恢复成为主障碍后转 Forensics。

## 完整调用约定

系统工具使用 WSL 绝对路径；数值脚本使用 `ctf-tools`；MCP 静态分析先检查会话。所有终端命令从 `pwsh` 发起：

```pwsh
wsl /usr/bin/file /path/to/firmware.bin
wsl /usr/bin/binwalk /path/to/firmware.bin
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools python /path/to/signal_analysis.py
wsl /usr/bin/qemu-arm -L /usr/arm-linux-gnueabihf/ /path/to/arm-binary
```

## 当前状态与路径

| 工具 | 当前状态/版本 | 路径或环境 | 何时使用 | 完整调用 |
|---|---|---|---|---|
| `file` | 可用，5.47 | `/usr/bin/file`，WSL system | 固件、镜像和未知载体首检 | `wsl /usr/bin/file /path/to/input` |
| `xxd` | 可用，Vim xxd 9.2.0524 | `/usr/bin/xxd`，WSL system | header、端序、寄存器表或帧字节检查 | `wsl /usr/bin/xxd -g 1 /path/to/input` |
| `binwalk` | 可用，2.4.3 | `/usr/bin/binwalk`，WSL system | 固件签名、嵌入结构和文件系统定位 | `wsl /usr/bin/binwalk /path/to/firmware.bin` |
| `numpy` / `scipy` | 可用，2.4.4 / 1.17.1 | WSL `ctf-tools` | 采样、滤波、相关、频谱和侧信道统计 | `wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools python /path/to/signal_analysis.py` |
| QEMU user | 可用，11.0.3 | `/usr/bin/qemu-arm`、`qemu-aarch64`、`qemu-mips`、`qemu-riscv64` | 已知架构的用户态程序重放 | 见上方 `qemu-arm` 示例；按架构改入口和 sysroot |
| IDA Pro MCP（`idalib`） | 方法可调用；当前 0 个会话；版本未暴露 | MCP | 已提取 firmware/boot code 的静态分析 | `idb_list` → `idb_open(绝对路径)` → `survey_binary` |
| Ghidra MCP | 方法可调用；当前无运行实例；版本未暴露 | MCP | IDA 不适用、需要显式 raw language 或已有项目时 | `list_instances` → `connect_instance`；raw 导入时再用明确 language 的 `import_file` |
| `sigrok-cli` / PulseView | 未发现 | WSL/Windows | 逻辑分析仪与 UART/I2C/SPI/CAN 协议解码 | 当前无可执行命令 |
| `rtl_433` / GNU Radio | 未发现 | WSL/Windows | RF/IQ 调制、同步和帧恢复 | 当前无可执行命令 |
| OpenOCD | 未发现 | WSL/Windows 与 `/home/kali` | JTAG/SWD 目标识别和调试 | 当前无可执行命令 |

## 失败处理

- `binwalk` 只有签名没有可复现提取：先手工记录 offset、压缩格式和文件系统，不用递归提取结果代替证据。
- QEMU 因 sysroot、动态链接器或 syscall 失败：先缩到单函数/用户态样本；完整固件环境再转 [firmware-loader-and-boot-chain-emulation.md](firmware-loader-and-boot-chain-emulation.md)。
- 逻辑/RF 工具缺失：保留原始 capture 和采样元数据，先用 `numpy/scipy` 做离线频谱与 framing；确认专用协议工具确实必要后再申请安装。
- MCP 没有会话/实例：这不是工具缺失；按表中顺序打开目标。文件架构和基址仍不明时先回到 `file`、header 和芯片资料。

## 关联知识页

- [signals-and-hardware.md](signals-and-hardware.md)
- [bus-logic-and-serial-frame-decoding.md](bus-logic-and-serial-frame-decoding.md)
- [rf-sdr.md](rf-sdr.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [firmware-loader-and-boot-chain-emulation.md](firmware-loader-and-boot-chain-emulation.md)
