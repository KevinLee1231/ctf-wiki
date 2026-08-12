# DownUnderCTF 2022 discord Writeup

## 题目简述

题面提示加入 DownUnderCTF Discord，在“spicy memes”中寻找由“certified memer”发布的内容。决定性证据不在附件算法中，而在比赛社区的公开频道、作者身份和帖子内容。

## 解题过程

进入比赛 Discord 后先查看与提示最吻合的 `#memes` 频道，再按题目作者 `NoSurf#3704` 搜索历史消息。该账号发布了一张澳洲主题 GIF，画面底部直接叠加了 flag；无需从 GIF 帧的像素或元数据中做额外隐写提取。

![NoSurf 在 DUCTF Discord memes 频道发布的澳洲主题 GIF，底部字幕直接显示比赛 flag](./DownUnderCTF2022-discord-wp/flag-meme.gif)

字幕内容为：

```text
DUCTF{G'day_mates_this'll_be_a_cracka}
```

## 方法总结

题面中的平台、频道主题和作者名共同构成检索条件：先限定官方 Discord，再限定 `#memes`，最后按发布者过滤，可避免在整个服务器逐条翻找。GIF 具有独立的梗图语境和视觉证据价值，因此保留原始载体并使用语义化文件名；flag 同时转写进正文，使归档不依赖未来仍能访问历史 Discord 消息。
