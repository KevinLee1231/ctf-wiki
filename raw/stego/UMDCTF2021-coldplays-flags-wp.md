# Coldplay's Flags

## 题目简述

题目给出一段音频。文件尾附加了两个 ZIP：一个包含时间戳提示，另一个受密码保护并保存 flag。密码需要根据提示时间点对应的歌曲歌词拼出。

## 解题过程

先检查文件尾和嵌入对象：

```bash
binwalk coldplay-audio
binwalk -e coldplay-audio
```

提取出的提示给出：

```text
(1:49)_(1:01)_(0:16)_(0:56)_(0:17)_(1:33)_a_(1:34)?
```

这些不是直接的数字密码，而是音频中的时间点。逐个跳转到对应位置，记录当时唱出的单词，再按提示中的下划线、固定字母 `a` 和末尾问号组合，得到 ZIP 密码：

```text
can_tchaikovsky_talk_to_skeletons_by_a_ouija?
```

用它解开第二个 ZIP：

```bash
unzip -P 'can_tchaikovsky_talk_to_skeletons_by_a_ouija?' flag.zip
```

其中内容为：

```text
UMDCTF-{PY07r_11Y1CH_7CH41K0V5KY}
```

## 方法总结

音频在本题同时充当宿主文件和语义线索。`binwalk` 负责发现追加 ZIP，时间戳则要求回到歌曲内容取词。外部歌曲信息不是解题的唯一载体：关键时间点、拼接规则和最终口令均已在这里展开，按照原音频即可独立复现。
