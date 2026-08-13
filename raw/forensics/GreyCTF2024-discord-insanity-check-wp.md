# GreyCTF2024 Discord Insanity Check WP

## 题目简述

题目把线索放在比赛 Discord 的一个隐蔽频道中：频道名提示检查 `gigacat` 表情的 PNG。flag 不是藏在像素中，而是以明文附加在 PNG 文件数据末尾，因此无需做视觉隐写分析。

## 解题过程

枚举比赛服务器的频道列表，会发现：

```text
gigacat-dot-png-sussy-baka
```

频道名指出应下载 Discord 表情 ID `1246722891941285980` 的 PNG 版本。取得文件后直接检查可打印字符串：

```bash
strings 1246722891941285980.png | grep -o 'grey{[^}]*}'
```

也可以从二进制尾部搜索 `grey{`。仓库保留的原始 PNG 会直接输出：

```text
grey{im_so_sorry_for_coming_up_with_this_challenge}
```

该表情的画面本身不参与推理，只有附加文本有价值，所以归档时把结果转写进正文，不保留低分辨率表情截图。

## 方法总结

PNG 解码器会忽略 `IEND` 之后的尾随数据，因此“能正常显示”不代表文件末尾没有附加内容。取证时先用 `file`、`strings`、十六进制搜索检查容器外围，再考虑像素位平面等更复杂方法。
