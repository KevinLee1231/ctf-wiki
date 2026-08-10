# Git Leakage

## 题目简述

题目站点把 Git 元数据目录直接暴露在 Web 根目录下。访问 `/.git/` 可以看到 `HEAD`、`objects`、`refs`、`index` 和 `COMMIT_EDITMSG` 等文件，说明服务端部署时误把完整仓库一并公开。

## 解题过程

先访问：

```text
http://challenge.example/.git/
```

目录可列出时，可以用 GitHack 一类工具恢复工作树，也可以手动下载 `HEAD`、引用、索引及对象文件后执行 Git 恢复。以 GitHack 为例：

```bash
CHALLENGE_BASE='http://challenge.example'
githack "$CHALLENGE_BASE/.git/" recovered
cd recovered
git status
git log --oneline --all
```

泄露的 `.git/COMMIT_EDITMSG` 提示最新提交添加了 flag 文件。恢复出的文件名为 `Th1s_1s-flag`，读取即可得到：

```text
hgame{Don't^put*Git-in_web_directory}
```

原 PDF 中的目录列表和终端截图都只承载文件名与命令输出，因此已转写为文本，不再重复保存截图。

## 方法总结

发现站点存在 `/.git/` 时，不能只查看当前目录中的普通文件；Git 对象库、索引和历史提交可能恢复出已删除或未部署到工作树的敏感内容。防御上应在部署阶段排除 `.git`，并由 Web 服务器显式拒绝所有点目录，而不是依赖目录索引关闭。
