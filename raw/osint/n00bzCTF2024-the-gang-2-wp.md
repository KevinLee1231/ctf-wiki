# The Gang 2

## 题目简述

从 John Doe 的战队资料继续调查。他发布的文章是一则谜语，每句话首字母组成隐藏指令，指向新的社交平台账号。

## 解题过程

按顺序提取谜语各句首字母，得到：

```text
USERNAMEISJOHNHACKERDOE
```

因此用户名为 `johnhackerdoe`。搜索该标识可定位到 X/Twitter 账号，显示名 `John Hacker Doe` 与人物一致。账号唯一一条帖子中包含：

```text
n00bz{5t0p_ch4s1ng_m3_4f2d1a7d}
```

## 方法总结

藏头文本要保留句子顺序与边界。得到用户名后，还应以显示名、上下文和帖子内容交叉确认，避免把同名账号误当作目标。
