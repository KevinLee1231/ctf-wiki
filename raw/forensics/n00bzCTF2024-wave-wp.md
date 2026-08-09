# Wave

## 题目简述

WAV 因文件头关键标识被破坏而无法正常播放。修复 RIFF/WAVE 结构后，音频内容是一段摩尔斯电码。

## 解题过程

在十六进制编辑器中修复三个位置：

```text
offset 0x00: RIFF
offset 0x08: WAVEfmt
offset 0x24: data
```

保存后播放器可以识别音频。根据长短声及间隔解码摩尔斯，得到连续小写文本：

```text
beepbopmorsecode
```

按题面补上 flag 外壳：

```text
n00bz{beepbopmorsecode}
```

## 方法总结

文件无法播放时应先核对魔数和区块标识，不要直接把静音当作隐藏信息。修复容器后再分析信号语义，能把“格式损坏”和“摩尔斯解码”两个阶段清晰分开。
