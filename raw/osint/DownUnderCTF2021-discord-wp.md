# DownUnderCTF 2021 - Discord

## 题目简述

题面把 `help`、Discord 和 `support` 连续作为提示。这是一道赛事平台导航题：flag 不在附件或远程程序中，而在 DownUnderCTF 2021 官方 Discord 的支持入口公开信息里。

## 解题过程

比赛期间从平台帮助页进入官方 Discord，并按帮助流程找到用于创建工单的 `#request-support` 频道。检查频道顶部的名称、主题和说明，而不是只翻阅聊天记录；flag 就写在该频道的描述中。

仓库中的正式验题配置 `challenge.yml` 给出的可接受值为：

```text
DUCTF{if_you_are_having_challenge_issues_come_here_pls}
```

需要注意，仓库附带的旧版 `WRITEUP.md` 把 `challenge` 误写成了复数 `challenges`。归档时应以实际判题配置中的单数形式为准。

## 方法总结

社交平台题中的“help”“support”“channel”等词通常是导航层级提示。检索时应依次核对官方入口、服务器身份、目标频道以及频道描述等静态元数据，并用判题配置或另一份官方材料交叉验证精确拼写；不要把临时比赛帮助页当成长期复现所必需的唯一信息。
