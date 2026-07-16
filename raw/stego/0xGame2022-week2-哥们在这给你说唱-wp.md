# week2哥们在这给你说唱

## 题目简述

附件是一段可正常播放的 WAV 音频，但同时叠加了 SilentEye 消息隐写和 DeepSound 文件隐写。DeepSound 能识别出音频中的隐藏文件，却要求输入密码；该密码正是另一层 SilentEye 隐写消息的内容。

## 解题过程

先不要直接爆破 DeepSound 的密码。将原始 WAV 载入 SilentEye，选择解码消息，得到：

```text
pass:15gmzzgnscltcltdz
```

随后把同一份 WAV 载入 DeepSound，选择提取隐藏文件，并输入密码 `15gmzzgnscltcltdz`。解出的文件中包含：

```text
0xGame{5d4d7df0-6de7-4897-adee-e4b3828978f8}
```

## 方法总结

这是一道双层音频隐写题：SilentEye 层保存的是下一层所需的密码，DeepSound 层才保存目标文件。遇到某个隐写工具能够识别载荷但要求未知密码时，应继续检查同一载体的其他隐写特征，而不是立刻穷举。WP 已记录工具分工、执行顺序和实际密码，不依赖外部教程也能复现解题链。
