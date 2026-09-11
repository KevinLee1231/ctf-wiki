---
type: tooling
tags: [hardware-embedded, tooling, tools, environment]
skills: [ctf-hardware-embedded]
---

# Hardware / Embedded Tooling

本页服务总线、物理接口、MCU 外设、RF/IQ、侧信道与启动信任链。普通固件逻辑恢复用 [reverse-tooling.md](reverse-tooling.md)，利用原语用 [pwn-tooling.md](pwn-tooling.md)。

## 平台与当前能力

离线逻辑采样、IQ、功耗 trace 和图像数组优先在 Windows 分析，Python 固定为 `D:/文档/新建文件夹/venv/Scripts/python.exe`（3.14.3）。固件来自 Linux 或 MCU，不等于解析数据必须进入 WSL。

| 入口 | 当前状态 | 用途 |
|---|---|---|
| Windows venv `numpy` / `scipy` | `2.4.4` / `1.17.1` | 样本读取、重采样、频谱、相关和基础信号处理 |
| Windows venv `matplotlib` | `3.11.0` | 波形、功率谱、时序和统计可视化 |
| Windows venv `capstone` / `unicorn` | `5.0.7` / `2.1.4` | CPU 指令与局部算法模拟；不自动提供 MCU 外设 |
| binwalk / unsquashfs | WSL `/usr/bin/binwalk`、`/usr/bin/unsquashfs`；APT `2.4.3+dfsg1-3` / `1:4.7.5-1` | 已有固件嵌入识别与文件系统提取工具 |
| QEMU user/system | WSL 已有，版本与调用见 Pwn 工具页 | 需要 Linux 用户态、内核或受支持机器模型时使用 |
| IDA/Ghidra | 当前静态入口见 Reverse 工具页 | 固件架构、端序、基址与代码/数据分界 |

本页没有已配置并验证的 USB 探针、JTAG/SWD、SDR 或逻辑分析仪连接。WSL 安装了模拟器也不能证明硬件透传、外设模型或加速可用；GNU Radio、sigrok、OpenOCD 等专项链不作为现成入口。先检查采集文件能否离线处理，再评估确需的硬件与软件。

## 完整调用

以下示例假定已知 IQ 是 little-endian complex64；真实格式必须从题面、采集设置或文件头确认：

```pwsh
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'import numpy as np; x=np.fromfile("D:/题目路径/capture.iq", dtype="<c8", count=4096); print(x.shape); print(np.abs(x[:8]))'
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/decode_trace.py"
wsl -d kali-linux --exec /usr/bin/binwalk "/mnt/d/题目路径/firmware.bin"
wsl -d kali-linux --exec /usr/bin/unsquashfs -s "/mnt/d/题目路径/rootfs.sqfs"
wsl -d kali-linux --exec /usr/bin/unsquashfs -d "/mnt/d/题目路径/rootfs-output" "/mnt/d/题目路径/rootfs.sqfs"
```

`D:/题目路径` 对应 `/mnt/d/题目路径`。先列嵌入段或超级块，再提取到题目输出目录；不默认递归运行所有提取器。QEMU 的目标 sysroot、machine、CPU、加载地址与设备应从题目配置确定，不能把现有二进制误当完整仿真环境。

## 失败处理与下一跳

- 波形/频谱没有稳定结构：核对采样率、符号率、端序、I/Q 排列与有符号格式；查 [signals-and-hardware.md](signals-and-hardware.md)、[rf-sdr.md](rf-sdr.md)。
- 总线帧解析不符：核对阈值、时钟边沿、bit order、起止位、校验与通道映射；查 [bus-logic-and-serial-frame-decoding.md](bus-logic-and-serial-frame-decoding.md)。
- 仿真卡在 MMIO、异常或启动早期：先定位缺失外设、ROM/RAM 映射和中断条件；查 [firmware-loader-and-boot-chain-emulation.md](firmware-loader-and-boot-chain-emulation.md)。
- 架构、启动模式或内核边界不明：查 [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)，不无目标地换模拟器。
- 缺包先评估 Windows 兼容版本。确需 WSL 新增、升级或重装时，必须先说明 Linux 必要性、安装方式、目标及依赖影响并取得明确同意。
