# Card Obsession

## 题目简述

题目给出一段约 12 秒的视频。画面本身没有承载有效信息，关键是音轨中的双音多频信号（DTMF）：每个按键由一对固定频率表示，连续按键之间的停顿还承担分组作用。

## 解题过程

先从视频提取音频，并用 DTMF 解码器识别按键：

```bash
ffmpeg -i VeryFunStuff.mp4 -vn card-obsession.wav
multimon-ng -a DTMF -t wav card-obsession.wav
```

结合音频波形中的停顿，可得到按键序列：

```text
88 6 3 222 8 333 444 3 666 66 8 555 444 55 33 222 2 777 3 7777
```

这不是电话号码，而是老式手机九宫格的多次按键输入法。同一组内的重复次数决定字母，例如 `88` 是 `U`、`6` 是 `M`、`3` 是 `D`、`222` 是 `C`。逐组解码得到：

```text
UMDCTFIDONTLIKECARDS
```

按 flag 格式整理为：

```text
UMDCTF-{i_dont_like_cards}
```

## 方法总结

多媒体题不能只看画面。先用 `ffprobe` 或 `ffmpeg` 确认各条流，再分别检查音频频谱和时间间隔。本题包含两层编码：DTMF 把声音变成数字，停顿又把数字切成手机多击键盘的字符组；若丢掉停顿，第二层边界就会消失。
