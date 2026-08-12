# DownUnderCTF 2020 - Bad Man

## 题目简述

题面只给出别名 `und3rm4t3r`，要求从公开来源追查该用户。用户名搜索能定位同名 Twitter 账号；其现存推文主动承认删除过一条泄露个人信息的帖子，因此关键证据不在当前页面，而在网页归档快照中。

## 解题过程

先对用户名做精确字符串搜索，并核对账号拼写、头像和推文语境。目标账号的现存推文内容是：

```text
whew that was close.... put out a tweet that contained personal information...
welp im glad we have a delete button
```

这条信息明确给出时间型 pivot：查询 [Internet Archive Wayback Machine](https://web.archive.org/) 中该账号 URL 的历史快照，而不是继续猜测其他平台。

在 2020 年 7 月 23 日的快照中，被删除的推文仍被归档，内容为：

```text
good job here is your flag:
DUCTF{w4y_b4ck_1n_t1m3_w3_g0}
```

官方 writeup 中两张图片只是上述纯文本推文的截图，关键信息已逐字转写，因此不保留 `Untitled.png`、`wayback.png` 这类弱命名截图。

## 方法总结

- 核心技巧：从唯一用户名定位公开账号，再利用自述“已删除”这一时间线信号查询历史网页快照。
- 识别信号：账号提及删除、误发、旧主页、改名或“以前发布过”时，应尽早检查 Wayback/搜索缓存，而不是只看当前页面。
- 复用要点：归档证据要记录目标 URL、快照日期和关键原文；同名账号必须通过内容、头像或交叉链接验证，不能因用户名碰撞就认定身份。
