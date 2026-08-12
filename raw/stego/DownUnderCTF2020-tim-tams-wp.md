# DownUnderCTF 2020 - Tim Tams

## 题目简述

题目给出 `Clive.wav`，听起来像模拟传真。它实际是一段 Slow-Scan Television（SSTV）传输：用音频频率逐像素发送静态彩色图像。文件采样率为 48 kHz、时长约 115.2 秒；VIS 码为十进制 44，对应 Martin M1 模式。图像解出后还包含一层 ROT13 文本。

## 解题过程

### 识别 Martin M1

音频开头符合 SSTV 的标准握手：1900 Hz leader、1200 Hz break、第二段 leader，随后是 30 ms 为一个 symbol 的 VIS 序列。按 LSB-first 解读 7 个数据位得到 $44=0x2c$，因此图像参数为 Martin M1：

| 项目 | 参数 |
| --- | --- |
| 图像尺寸 | $320 \times 256$ |
| 每行同步 | 1200 Hz，4.862 ms |
| porch | 1500 Hz，0.572 ms |
| 通道顺序 | Green、Blue、Red |
| 每个通道扫描 | 146.432 ms |
| 通道分隔 | 1200 Hz，0.572 ms |
| 像素亮度范围 | 1500 Hz 为 0，2300 Hz 为 255 |

可把 WAV 直接交给支持 SSTV 的解码器并选择 Martin M1。为了核对原始附件，本次还用 Hilbert 变换取得瞬时相位，在每个像素时间窗内求平均频率，再按下式映射为通道值：

$$
v=\operatorname{clip}\left(\frac{f-1500}{2300-1500}\times255,\ 0,\ 255\right)
$$

每行依次写入 G、B、R 三个通道，连续处理 256 行后得到原图：

![Martin M1 解调出的 Clive Palmer 海报，左上角包含 ROT13 flag 文本](DownUnderCTF2020-tim-tams-wp/sstv-clive-message.png)

图像左上角的文本为：

```text
QHPGS{UHZOYR_Z3Z3_1BEQ}
```

### ROT13 解码

字母部分按 ROT13 旋转，数字和符号保持不变：

```python
import codecs

print(codecs.decode("QHPGS{UHZOYR_Z3Z3_1BEQ}", "rot_13"))
```

得到：

```text
DUCTF{HUMBLE_M3M3_1ORD}
```

## 方法总结

- 核心技巧：从模拟音频的 leader/VIS 序列识别 SSTV 模式，按 Martin M1 的同步、通道顺序和频率映射恢复图像，再处理图中文字编码。
- 识别信号：持续一两分钟、带固定高低音和周期同步脉冲的“传真声”应考虑 SSTV；开头 VIS 码比盲试不同模式更可靠。
- 复用要点：解调时先锁定采样率、VIS、行起点和 skew；Martin M1 是 G-B-R 顺序而不是 RGB。恢复图像后仍要检查文本是否有 ROT、Base64 等表示层编码。
