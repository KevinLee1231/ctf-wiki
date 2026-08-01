# BYUCTF 2023 - National Park

## 题目简述

附件是一个包含 7635 个文件的游戏包。题面提示进入野生宝可梦战斗时音频异常，目标文件是 `National Park/Audio/BGM/Battle wild.ogg`。

## 解题过程

实际运行游戏可以触发异常，也可以按修改时间和文件名直接定位音频。波形在约 60–80 秒出现明显的短、长脉冲组：

![Battle wild.ogg 在 60 至 80 秒范围内的长短脉冲](./BYUCTF2023-national-park-wp/wild-battle-morse-pulses.png)

按持续时间把短脉冲记为点、长脉冲记为划，字符间和单词间用较长静音分隔；对完整音轨而非上面的局部示例逐组转写，再交给 Morse 解码器，得到：

```text
BYUCTFR0M_HACXS_RUL3
```

按题意补上花括号：

```text
BYUCTF{R0M_HACXS_RUL3}
```

## 方法总结

大型游戏资源包不应无差别分析。题面已经把触发事件限定为“野怪战斗”，先按资源路径和时间戳缩小到对应 BGM，再用完整波形的持续时间识别 Morse；局部截图只用于展示点划形态，不能替代全段转写。
