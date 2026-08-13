# Someone's Salty

## 题目简述

附件聊天截图给出用户名 `ducati777__`、头像以及“flag 来自 GitHub 仓库”“dig it up”等提示。目标是沿公开身份关联从 GitHub 转到 Mastodon，再寻找账号迁移前的旧帖子。

## 解题过程

初始聊天截图只是文字界面，决定性信息可直接转写为：

```text
显示名称：Mark Dafydd
用户名：ducati777__
线索：有人从其 GitHub 仓库偷走 flag，并让参赛者把它“dig it up”。
```

搜索近似用户名可找到 GitHub 账号 `ducati777-o`。头像与聊天账号一致，个人简介又写明有人从其仓库偷走 flag，因此不是仅凭名称相似做关联。GitHub 资料页同时公开了下一跳：

```text
Mastodon：@ducati777@mastodon.social
```

继续检查该 Mastodon 身份的帖子和迁移信息，会发现内容指向旧服务器账号 `ducati777@defcon.social`。访问旧账号的历史帖子即可得到：

```text
grey{d169InG_4_fOs5iLi23d_Fl4g}
```

旧账号公开页为 [defcon.social/@ducati777](https://defcon.social/@ducati777)。正文已保留完整的身份关联链，链接只作为原始公开证据入口。

## 方法总结

- 核心技巧：用用户名、头像、简介语义和跨平台链接建立多项一致的身份链，再追踪 Mastodon 账号迁移。
- 识别信号：题面使用 “dig it up” 暗示旧内容；GitHub 简介不仅给出下一平台，还明确呼应“被偷 flag”的上下文。
- 复用要点：跨平台账号不能只靠同名认定；至少应组合头像、个人简介、互链或迁移声明中的两项以上证据。
