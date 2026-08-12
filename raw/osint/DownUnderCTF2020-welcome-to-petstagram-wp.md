# DownUnderCTF 2020 - Welcome to Petstagram

## 题目简述

题目只给出 “Alexandros the cat”、它口中的 “mum” 以及 “Petstagram” 三个线索，要求提交 mum 的完整姓名。Petstagram 暗示入口是 Instagram；难点是从宠物账号沿关注关系、跨平台用户名、邮箱命名和自动回复逐步确认同一人的中间名。

## 解题过程

### 从宠物账号找到主人

在 Instagram 搜索 `alexandrosthecat`，可以定位 Alexandros 的账号。它最早的关注者之一是 `emwaters92`，显示名为 Emily Waters；该账号同时发布过 Alexandros 的照片，个人简介还给出邮箱：

```text
emilytwaters92@gmail.com
```

这些证据足以把猫和 Emily Waters 关联起来，但题目要求 full name，邮箱里的 `t` 表明还缺一个以 T 开头的中间名。

### 跨平台确认中间名

Alexandros 的帖子包含短链 `bit.ly/3h3e7A0`，指向用户名 `gelato_elgato` 的 YouTube 频道。继续搜索同一用户名，可以在 Twitter 找到由 Emily 使用的账号，其简介为 “love gelato and my cat alexandros”，与 Instagram 的人、猫和兴趣三项同时匹配。

Twitter 显示名 “call me theresa” 给出中间名候选 Theresa。为了避免仅凭昵称猜测，向公开简介中的 Gmail 地址发信后，题目设置的 out-of-office 自动回复署名为：

```text
Kind Regards,
Emily Theresa Waters.
```

自动回复还再次链接 Alexandros 的 Instagram，因此形成闭环验证。按小写、空格改下划线的格式提交：

```text
DUCTF{emily_theresa_waters}
```

## 方法总结

- 核心技巧：从宠物账号的社交关系出发，通过复用用户名、简介内容、邮箱模式和自动回复验证现实身份字段。
- 识别信号：题面用仿平台名提示入口，邮箱本地部分含额外首字母，而其他平台显示名给出候选中间名时，应继续寻找独立证据确认。
- 复用要点：跨平台关联至少需要两个以上稳定属性，不能只凭同名；向公开邮箱发送消息属于主动交互，真实调查中必须确认授权，本题则由官方专门配置自动回复作为证据。
