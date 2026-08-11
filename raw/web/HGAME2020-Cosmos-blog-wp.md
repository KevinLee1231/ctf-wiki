# Cosmos 的博客

## 题目简述

题目给出一个博客站点，并用“版本管理工具”和 GitHub 暗示源码仓库可能被直接部署到了 Web 根目录。关键风险是站点暴露了 `.git` 元数据：即使 flag 文件已经从当前版本删除，提交历史和远端仓库配置仍可能保留它。

## 解题过程

先访问 Git 的引用文件确认泄露：

```text
/.git/HEAD
```

如果能读到类似 `ref: refs/heads/master` 的内容，就可用 GitHack 一类工具递归恢复 `.git` 目录。恢复后进入工作区检查提交与远端配置：

```bash
git status
git log --all --stat
git remote -v
```

本地当前版本没有 flag，但 `git remote -v` 暴露了 GitHub 仓库地址。取得远端历史后继续搜索被删除文件：

```bash
git log --all --diff-filter=D --summary
git show <删除前的提交>:<文件路径>
```

历史文件中保存的是一段 Base64 文本，对其解码即可恢复：

```bash
printf '%s' '<base64-data>' | base64 -d
```

得到：

```text
hgame{g1t_le@k_1s_danger0us_!!!!}
```

## 方法总结

- 核心技巧：从公开的 `.git` 目录恢复仓库，再沿本地配置定位远端并检查历史删除记录。
- 识别信号：站点出现版本管理、GitHub、提交等提示，或 `/.git/HEAD`、`/.git/config` 可以直接访问。
- 复用要点：不能只检查当前工作树；`git log --all`、删除记录、旧提交和远端引用都可能保存敏感信息。
