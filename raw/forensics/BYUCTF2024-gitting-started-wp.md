# Gitting Started

## 题目简述

题目给出一个公开博客仓库。当前工作树中已经找不到 flag，但 Git 对象数据库仍保存曾经提交、后来删除的内容，目标是审查完整提交历史。

## 解题过程

克隆题目给出的 [tech-blog 仓库](https://gitlab.com/TheITFirefly/tech-blog)，不要只搜索检出的最新版本：

```text
git clone https://gitlab.com/TheITFirefly/tech-blog.git
cd tech-blog
git log --all -p -- .
```

`-p` 会显示每个提交的补丁，包括被删除的行。可进一步在历史补丁中搜索比赛前缀：

```text
git log --all -p | grep -i 'byuctf{'
```

删除记录中出现：

```text
byuctf{g1t_gud!}
```

## 方法总结

公开仓库的“当前文件”只是一个快照。敏感内容只要进入过提交，就可能继续存在于历史、分支、标签或悬空对象中；审计时至少覆盖 `git log --all -p`，清密钥则必须轮换凭据，不能只提交一次删除。
