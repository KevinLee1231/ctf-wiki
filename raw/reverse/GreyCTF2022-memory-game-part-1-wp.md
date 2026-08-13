# GreyCTF2022 - Memory Game Part 1

## 题目简述

题目给出 Android APK，并提示 flag 与可绘制资源有关。真正的 flag 没有经过算法加密，而是作为 PNG 资源直接打包进 APK，决定性步骤是解包并检查资源，而非通关游戏。

## 解题过程

APK 本质上是 ZIP 容器，可用 Apktool 保留资源名地反编译：

```bash
apktool d memory-game.apk -o memory-game
find memory-game/res -type f -iname '*flag*'
```

结果指向 `res/drawable-hdpi/flag.png`。打开该资源即可看到一行 flag：

```text
grey{th1s_1s_dr4w4bl3_bu7_e4s13r_t0_7yp3}
```

仓库还保留了用于制作该资源的 `flag.xcf`。逐层检查后确认其中仅有相同的文本行和白色背景，不包含布局、像素关系或其他必须靠视觉理解的信息，因此题解直接转写文本，不归档一张重复的“文字截图”。

## 方法总结

Android 逆向应先盘点 Manifest、`res/`、`assets/` 和原始资源，再决定是否深入 Java/Smali。资源名被保留时，语义化文件名可能直接暴露目标；即便名称被混淆，也可按尺寸、类型和引用关系批量缩小范围后逐张查看。
