---
type: family
tags: [forensics, family, usb, hid, bluetooth, peripheral]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/peripheral-capture.md
updated: 2026-06-12
---

# Peripheral Capture Analysis

## 作用边界

本页是外设流量和输入设备取证 family，覆盖 USB HID 键盘/鼠标/手写笔、LED output report、方向键路径、Bluetooth RFCOMM、GBA USB interrupt framebuffer 等。它负责把 PCAP/usbmon/蓝牙/URB 记录恢复成按键、轨迹、帧缓冲或交互序列。

边界在于“协议/report 解析”：如果原始证据是视频、音频、物理运动轨迹，或者已经从 capture 中提取出按键/坐标序列，后续读图、读频谱、读动作应转 [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)。

## 识别信号

- 捕获中出现 USB URB interrupt、HID report、Bluetooth RFCOMM、键盘 scan code、鼠标相对坐标、LED output report 或手写笔坐标。
- flag 可能由按键序列、鼠标绘制图、方向键路径、LED 闪烁或设备 framebuffer 表示。
- Wireshark 能看到设备类，但通用文本提取无结果。

## 最小证据

- 设备类型、report descriptor 或至少 report 长度和字段位置。
- 时间顺序、按下/释放状态、modifier、坐标增量、LED 状态或帧缓冲分片。
- 能从一小段 capture 手工复原几个字符/坐标，确认解析方向。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| USB keyboard HID | report byte、modifier、keycode 和 release 事件 | 映射 keycode，处理 shift/caps |
| USB mouse/pen | X/Y 相对或绝对坐标、button 状态 | 累加轨迹并绘图 |
| LED output report | Caps/Num/Scroll 状态变化时间 | 先提取 on/off 序列；若只剩节奏/视频读码，转 [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md) |
| Arrow key navigation | 方向键序列和起点/地图 | 先恢复输入序列；若需要重放地图或画图，转 [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md) |
| Bluetooth RFCOMM | L2CAP/RFCOMM 重组和 payload 方向 | 提取串口文本或文件 |
| GBA/USB framebuffer | interrupt payload 是否为像素/瓦片数据 | 按尺寸和色深重组图像 |

## 合并与拆分结论

- 保留为 family：外设 capture 的共同点是 report/帧/事件重组，而不是某个单一协议。
- 不合并进 `signals-and-hardware.md`：硬件页处理物理采样和总线，本页处理已捕获的外设协议记录。
- 不合并进 `keyboard-mouse-audio-and-physical-puzzles.md`：本页解决 report 字段、方向、release 和重组；对方解决动作轨迹、频谱、视频和物理时序的语义还原。
- 不拆键盘/鼠标/蓝牙小页：当前 raw 规模适合集中作为外设二级分流。

## 常见误判

- 不看 release 事件，导致重复字符或 modifier 错位。
- 鼠标坐标未累加或方向反了，绘图无法辨认。
- 蓝牙流没按方向重组，payload 顺序错。
- 只导出 pcap 文本，忽略 HID report descriptor。

## 关联页面

- [signals-and-hardware.md](signals-and-hardware.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [peripheral-capture.md](../raw/forensics/peripheral-capture.md)
