# Grey Yuumi

## 题目简述

题目附件由 League of Legends 回放（`.rofl`）和 USB capture（`.pcapng`）组成。回放中角色阵亡的时刻为鼠标涂写提供了时间锚点；USB HID capture 保存的是相对鼠标位移和按键状态。核心不是游戏逆向，而是将两个时间轴对齐、提取 HID 事件并重建被右键按住时的轨迹，因此归为 forensics。

已知回放总时长为 `26:23.925`，关键死亡计时器为 `5:36`、`7:02`、`9:40`、`12:40`、`18:14`、`21:21`、`23:09`、`24:17`、`26:09`。

## 解题过程

### 以回放时间定位 PCAP 窗口

PCAP 的相对时长不一定等于游戏时长，不能把两个时间戳直接相等比较。令 $T_g$ 为 `26:23.925` 的秒数，$T_p$ 为首尾 HID event 的 PCAP 时间差，对每个阵亡时刻 $t_g$ 使用线性映射：

$$
t_p=t_{p,0}+\frac{t_g}{T_g}T_p.
$$

围绕每个 $t_p$ 取前后 20 秒的窗口。官方 solver 据此处理九段候选，而不依赖两个录制过程恰好同步。

### 解码 USB 鼠标报告并绘制

使用 `tshark` 提取 `usbhid.data` 与 `frame.time_relative`。本题主要报告格式是：

```text
button = report[0]
dx     = signed little-endian int16(report[2:4])
dy     = signed little-endian int16(report[4:6])
wheel  = signed int8(report[6])
```

官方脚本还兼容四字节的 8-bit 相对位移报告。对每个报告累加位移：`x += dx`、`y += dy`。只保留 `button & 0x02 != 0` 且位移非零的线段，在每一个时间窗口按前一坐标到后一坐标连线；九段右键笔画组合后读出 flag。

```bash
tshark -r grey_yuumi.pcapng -Y usbhid.data -T fields \
  -e frame.time_relative -e usbhid.data
```

官方 `solve.py` 会解包题目压缩包、完成时间换算、输出每个候选窗口的 PNG。其绘图逻辑的关键是保留持续右键按下的相邻 HID 段，而不是把每个绝对坐标画成孤立点。

### 验证

重建出的笔画文本为：

```text
grey{yuum1logg3r_4ttach3d}
```

## 方法总结

- 核心技巧：利用独立载体中的事件时间作为 PCAP 的时间锚点，再从 HID 相对位移积分恢复用户动作。
- 识别信号：PCAP 出现 `usbhid.data`，题面又给出回放、录屏或其他有明确事件时刻的媒体时，应优先考虑跨载体时间对齐。
- 复用要点：HID 位移是相对值，要从统一原点累积；时间轴来源不同则应按总时长比例映射，并以按键位标志筛选真正的绘制动作。
