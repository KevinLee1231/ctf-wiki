---
type: technique
tags: [hardware-embedded, forensics, logic-analyzer, uart, spi, i2c]
skills: [ctf-hardware-embedded, ctf-forensics]
raw:
  - ../raw/hardware-embedded/signals-and-hardware.md
updated: 2026-07-27
---

# Bus, Logic and Serial-Frame Decoding

## 适用场景

逻辑分析、UART/I2C/SPI/CAN/显示链路采样中承载命令、按键、固件或秘密；需先恢复电平、时钟、帧边界和端序，再解析上层协议。

## 识别信号

- 数字波形有稳定时钟、起止位、片选或差分帧。
- 多通道边沿关系符合 SPI/I2C/UART/CAN。
- 固定 header、长度、CRC 或命令码周期出现。

## 最小证据

- 记录采样率、通道映射、逻辑阈值、波特率/时钟模式。
- 至少一帧可由协议字段/CRC 验证。
- 解码字节能与设备行为或已知命令对应。

## 解法骨架

1. 可视化边沿与周期，估计 bitrate/clock polarity/phase。
2. 按协议切帧并测试端序、bit order 和采样边沿。
3. 用 header/length/CRC 锁定参数，处理丢边沿和毛刺。
4. 将 payload 交给上层命令、文件或状态解析。

## 关键变体

- UART async serial。
- SPI/I2C 同步总线。
- CAN、显示链路或自定义 framing。

## 常见陷阱

- 采样率不足导致边沿混叠。
- SPI mode/bit order 选错仍产生可打印假数据。
- 只解码字节，不验证 CRC/设备语义。

## 关联技巧

- [signals-and-hardware.md](signals-and-hardware.md)
- [hardware-isa-bootloader-and-kvm.md](hardware-isa-bootloader-and-kvm.md)
- [rf-sdr.md](rf-sdr.md)

## 原始资料

- [signals-and-hardware.md](../raw/hardware-embedded/signals-and-hardware.md)
