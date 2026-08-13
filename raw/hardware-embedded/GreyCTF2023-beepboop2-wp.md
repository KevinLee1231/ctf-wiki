# GreyCTF2023 BeepBoop2

## 题目简述

附件是由 GNU Radio 流图生成的 QPSK 音频信号。仓库同时保留了 `.grc` 与生成的 Python 文件，明确给出采样率 32000、每符号 4 个采样点、QPSK 星座和接收链，因此无需靠听感猜编码。

## 解题过程

在 GNU Radio Companion 中打开 `mpsk_stage6_release.grc`，把 WAV 文件接入接收链。有效处理顺序为：

```text
WAV source
-> frequency correction
-> root-raised-cosine filtering
-> symbol clock recovery
-> linear equalizer
-> Costas loop
-> QPSK constellation decoder
-> differential decoder
-> bit repack (1 -> 8, MSB first)
-> file sink
```

流图中的关键参数保持为：

```text
sample rate = 32000
samples per symbol = 4
frequency offset = 0.25
output = /tmp/flagout.txt
```

运行后先让定时恢复与 Costas 环收敛，跳过接收开头的瞬态字节，再查看文件输出。恢复文本为：

```text
grey{n1c3_c0nst3ll4t10ns_3204390u3294qpskdi3d}
```

## 方法总结

数字调制题应按“载波、符号时钟、星座映射、差分编码、比特打包”的层次逐步还原。题目附带的流图已经给出了发射端与接收端的真实参数，比从频谱截图猜测可靠得多。星座看似稳定仍不代表字节正确，差分解码方向和 MSB/LSB 打包顺序同样必须与生成端一致。
