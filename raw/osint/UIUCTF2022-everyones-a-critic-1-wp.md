# Everyone's A Critic 1

## 题目简述

题目要求从 UIUCTF 官方 Discord 中找到一名游戏评测内容创作者。已知此人在服务器内唯一可识别，flag 的正文以 `th` 开头；本题也是后续五题所用虚拟身份的起点。

决定性线索不是泛搜“游戏评测”，而是题面明确限定了 UIUCTF Discord。先在这个封闭范围内确定账号，再沿用户名和头像跨平台关联，能显著减少撞名造成的误判。

## 解题过程

### 1. 在 Discord 内缩小范围

在 UIUCTF Discord 的全局搜索中检索 `review`、`reviewing` 等与评测直接相关的词。历史搜索结果同时显示账号头像、用户名、频道和三条相关发言；用户 `chuck.lephucke#0038` 明确表示自己将评测 UIUCTF。

![Discord 全局搜索中 chuck.lephucke 关于评测 UIUCTF 的三条消息](./UIUCTF2022-everyones-a-critic-1-wp/discord-review-search.png)

账号名、发言语义与题面完全吻合，所以可以把 `chuck.lephucke` 作为后续跨平台检索的稳定标识。此处应记录完整用户名和头像，而不只记录显示名；后者更容易被其他账号复用。

### 2. 检查账号资料与角色

打开该用户的 Discord 资料卡。比赛期间，flag 被放在用户的服务器角色名称中；公开留存的截图里可以看到早期版本：

![Discord 资料卡中的用户名、YouTube 线索和早期 flag 角色](./UIUCTF2022-everyones-a-critic-1-wp/discord-profile-role.png)

截图显示的是较短的 `uiuctf{this_flag_is_not_bait}`，而最终公开仓库的 `osint/critic-1/challenge.yml` 记录为：

```text
uiuctf{this_flag_is_not_bait_im_serious_its_not_just_trust_me_bro}
```

这是历史截图与最终题目元数据之间的版本差异，不能把短截图悄悄改写成长 flag。本文以官方最终元数据作为提交答案，同时保留截图所能证明的原始状态。完整检索路径也可在[参赛者留存的图文证据](https://github.com/silly-lily/CTF-Writeups/tree/main/2022_UIUCTF/osint/EveryonesACritic1)中交叉核对。

## 方法总结

本题的核心是“限定平台内搜索，再固定身份”。先利用题面把搜索范围收敛到官方 Discord，通过与 `review` 相关的发言锁定 `chuck.lephucke#0038`，再检查其资料卡和服务器角色获得 flag。后续跨平台追踪时，应同时比对用户名、头像和内容主题，不能仅凭一个相似昵称认定账号归属。

最终 flag：

```text
uiuctf{this_flag_is_not_bait_im_serious_its_not_just_trust_me_bro}
```
