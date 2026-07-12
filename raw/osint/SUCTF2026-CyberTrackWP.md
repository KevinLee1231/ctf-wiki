# SUCTF2026-CyberTrack

## 题目简述

这是 OSINT/社工轨迹题。题面入口是一个安全爱好者个人博客，要求从公开站点、GitHub、邮箱头像、Minecraft 历史昵称、社交平台和 Discord 线索中还原两部分信息：真实姓名和 `string`。题目实际检查的不是某个服务漏洞，而是身份链路能否闭合；flag 规则把“姓在前、名在后”的姓名与 `string` 拼接后做小写 MD5。

这题有两个解法分支：

- 非预期捷径：GitHub commit patch 泄露了邮箱 `evanlin1123@foxmail.com`，给邮箱发信可以收到自动回复，从而直接拿到姓名。
- 出题人预期：先从博客和头像识别到 Gravatar，利用 Gravatar 头像 URL 中的 SHA-256 邮箱哈希，结合公开昵称、生日、猫名、Minecraft 用户名等信息生成候选邮箱并爆破出 `evanlin1123@foxmail.com`。

`string` 分支主要依靠博客文章中的 Minecraft 线索。题目给出 `Mnzn233` 这个 MC 用户名后，可通过 NameMC 找到历史昵称，再把历史昵称拿去社交平台检索，最终定位到 Twitter/X 上的 Discord 邀请链接，并在 Discord 自我介绍中拿到 `string`。

## 解题过程

先从题面给出的博客入口整理可用信息。博客里的文章并不是都直接给答案，但它们提供了构造身份候选的高价值信号：

```text
Don't spam      -> evanlin1123@foxmail.com
Happy birthday  -> 2024-11-23
Today           -> Momo 是一只布偶猫
Play with me_t_t -> MC 用户名 Mnzn233
```

现有题解里使用了一个更直接的非预期路径：从博客跳到 GitHub 仓库，查看提交记录的 patch：

```text
https://github.com/EvanLin-SUCTF/EvanLin-SUCTF.github.io/commit/2796f3b4537dc0c1891da002dc9d02ab9f71b008.patch
```

patch 中暴露了邮箱 `evanlin1123@foxmail.com`。向这个邮箱发送邮件后会收到自动回复，自动回复中包含真实姓名 `Zeyuan Lin`。因为题目要求姓名拼接时姓在前、名在后，所以用于最终计算的是 `LinZeyuan`。

出题人预期路径不依赖 commit 泄露。博客头像使用 Gravatar，头像链接本身包含邮箱小写后的 SHA-256 哈希。由题面和博客可收集的关键词包括：

```text
EvanLin
EvanLin1123
Mnzn233
Momo
Ragdoll
foxmail.com
20241123 / 1123
```

把这些关键词组合成邮箱候选，统一转小写后计算 SHA-256，与 Gravatar 哈希比较。命中后即可得到同一个邮箱 `evanlin1123@foxmail.com`，再走邮件自动回复拿姓名。

`string` 的获取路线如下。先根据 `Play with me_t_t` 里的 Minecraft 用户名 `Mnzn233` 查询 NameMC：

```text
https://namemc.com/profile/Mnzn233.1
```

NameMC 能看到历史昵称，其中 `TurbidCloud` 是后续检索的关键。继续用这个历史昵称在社交平台搜索，能找到关联的 Twitter/X 账号，并看到一个 Discord 邀请链接。

![NameMC 和社交平台线索](SUCTF2026-CyberTrackWP/namemc-和社交平台线索.png)

进入 Discord 后查看用户自我介绍，可以看到 `Flag@here` 一类提示和最终 `string = ddc7-4622-8a97`：

![Discord 线索](SUCTF2026-CyberTrackWP/discord-线索.png)

![Discord 自我介绍](SUCTF2026-CyberTrackWP/discord-自我介绍.png)

因此整条链路可以概括为：

```text
博客入口
  -> GitHub / Gravatar 邮箱哈希
  -> 邮箱自动回复
  -> 真实姓名 Zeyuan Lin

博客 Minecraft 线索
  -> Mnzn233
  -> NameMC 历史昵称 TurbidCloud
  -> Twitter/X
  -> Discord
  -> string = ddc7-4622-8a97
```

## 方法总结

- 核心技巧：OSINT 身份关联、邮箱哈希还原、跨平台昵称追踪。
- 识别信号：题面给出个人博客、头像、GitHub 链接、生日、昵称、游戏平台用户名时，应优先建立“邮箱/昵称/生日/头像哈希”的候选图。
- 复用要点：GitHub commit patch、Gravatar 哈希、邮件自动回复、NameMC 历史昵称和社交平台邀请链接都可能成为身份闭环证据。写 WP 时应保留能复现关联链的 URL、关键昵称和验证点，而不是只写最后的 flag。
