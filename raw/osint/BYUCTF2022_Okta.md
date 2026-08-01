# BYUCTF 2022 - Okta

## 题目简述

题目要求从 2022 年 LAPSUS$ 泄露到社交媒体的私聊截图中补全一句话：“most of the time if you don't do anything like ____ ______, you won't be detected”，并附上截图时间戳。

## 解题过程

使用 `LAPSUS$ Okta leaked chat most of the time won't be detected` 等原句片段检索 2022 年 3 月的 Twitter 转发截图。官方记录保留的其中一份来源是 [John Hammond 的截图帖](https://twitter.com/_JohnHammond/status/1506166671664463875)。原句完整内容为：

```text
most of the time if you don't do anything like port scanning, you won't be detected
```

同一消息气泡旁的时间为 `11:22`。题面要求两词以下划线连接，再追加时间：

```text
byuctf{port_scanning_11:22}
```

旧官方说明提到：部分转发者的截图因设备时区不同而显示调整后的时间，比赛期间也接受了相应变体。本题归档采用官方主截图的 `11:22`。

## 方法总结

搜索泄露聊天时，完整引语片段比组织名称更有区分度。答案包含截图界面的时间，而不是报道发布时间；遇到多份转发截图还要检查时区差异，并以题目认可的主证据为准。
