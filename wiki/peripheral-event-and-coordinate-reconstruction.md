---
type: technique
tags: [forensics, hardware-embedded, usb-hid, mouse, keyboard, bluetooth]
skills: [ctf-forensics, ctf-hardware-embedded]
raw:
  - ../raw/forensics/peripheral-capture.md
updated: 2026-07-27
---

# Peripheral Event and Coordinate Reconstruction

## 适用场景

USB HID、鼠标、键盘、Bluetooth RFCOMM、LED 或其它外设 report 记录用户输入；需按 descriptor/协议解释按键状态、相对坐标和事件时序，重建文本、轨迹或符号。

## 识别信号

- PCAP/USBPcap 中有固定长度 interrupt report。
- 报文含 modifier、keycode、button、delta X/Y 或重复状态。
- 输入事件累积后可形成文字、绘图、Morse 或手势。

## 最小证据

- 确认 endpoint、方向、report descriptor/布局和坐标符号位。
- 区分 key down/up、modifier 与重复报告。
- 重建输出可由事件回放、轨迹图或已知按键验证。

## 解法骨架

1. 按设备/endpoint/会话过滤并按时间排序。
2. 解析 report 字段，维护键盘状态或累积鼠标坐标。
3. 处理布局、Shift/Caps、相对/绝对坐标和分段。
4. 输出按键日志/轨迹图，并回到原包号验证异常点。

## 关键变体

- USB keyboard/mouse HID。
- Bluetooth RFCOMM/HID。
- LED/Morse/特殊控制器事件。

## 常见陷阱

- 把 key-up/重复包当新按键。
- 相对坐标按无符号解释。
- 忽略键盘布局和 modifier 状态。

## 关联技巧

- [peripheral-capture.md](peripheral-capture.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [protocol-stream-reassembly-and-credential-extraction.md](protocol-stream-reassembly-and-credential-extraction.md)

## 原始资料

- [peripheral-capture.md](../raw/forensics/peripheral-capture.md)
