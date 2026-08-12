# DownUnderCTF 2022 This Takes Me Back Writeup

## 题目简述

题目给出历史事件：DUCTF 团队在 2021 年 9 月 28 日于 Sydney 聚会，并在午夜被请离，要求找出当时 Triple J 正在播放的歌曲。当前“最近播放”页面只展示近实时内容，因此必须查询历史网页，而不是在现站直接搜索。

这里的“午夜”是 Sydney 当地时间，并位于 9 月 28 日结束、9 月 29 日开始的边界。检索时需要同时检查边界两侧，避免因 UTC 与澳大利亚当地时区换算而错一天。

## 解题过程

官方题解给出的关键证据是 Internet Archive 保存的 [2021-09-28 Triple J Recently Played 页面快照](https://web.archive.org/web/20210928183228/https://www.abc.net.au/triplej/featured-music/recently-played/)。快照保留了当时页面及日期、时段筛选入口；将日期设置为 2021 年 9 月 28 日，并查看 Sydney 当地午夜附近的播放记录。

跨过日期边界的对应记录显示歌曲标题为：

```text
Ain't It Fun
```

题目要求歌名去掉空格，且接受省略撇号的形式，因此可提交：

```text
DUCTF{AintItFun}
```

## 方法总结

历史页面 OSINT 的重点是时间语义和证据固化。先明确事件所在地时区，再围绕午夜边界检查前后记录；动态网页失去历史数据后，应使用指定时间的网页快照。WP 不能只保存 Wayback 链接，还要写明目标日期、当地时区、查询位置和最终记录，保证读者不打开外链也能理解结论。
