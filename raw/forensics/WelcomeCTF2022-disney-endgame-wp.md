# Disney Endgame

## 题目简述

附件包含内容几乎相同的 `original.srt` 与 `altered.srt`。修改版在字幕文字、序号和时间戳中有规律地插入单个字符；按文件顺序收集这些新增字符即可重建 Flag。

## 解题过程

先做逐词差异比较，避免完整行差异掩盖真正改动：

```bash
git diff --no-index --word-diff=plain original.srt altered.srt
```

开头几处差异为：

```text
grade.   -> ggrade.   # 新增 g
grand    -> grrand    # 新增 r
Dale,    -> Dalee,    # 新增 e
Ugly     -> Uglyy     # 新增 y
months   -> monthhs   # 新增 h
```

后续改动也遵循同一规则，但载体不只限于台词。例如时间戳 `00:28:13,333` 被改成 `00:28:113,333` 时应取新增的 `1`；箭头、字幕编号和标点中的改动也要纳入。按文件自上而下只记录每处新增字符，拼接得到：

```text
greyhats{uG1y_50n1C_g0E5_S1oW_B4bY!_1bf80d36a}
```

## 方法总结

本题考查对两个数字证据版本的差异恢复。SRT 的时间轴、编号和正文都是有效字段，不能只比较台词。面对“每处多一个字符”的载荷，应记录插入字符而不是整段替换文本，并保持原文件顺序；最终字符串的 Flag 格式还能用来发现漏掉的时间戳或标点差异。
