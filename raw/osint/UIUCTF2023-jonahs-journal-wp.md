# UIUCTF 2023 Jonah's Journal Writeup

## 题目简述

题目说明 Jonah 把旅行笔记“推送”到线上 notebook，并提示用户名较为一致，同时使用 `forks`、`trees`、`pushing`、`pulling` 等词暗示 Git/GitHub。目标是找出他下一站的国家。

## 解题过程

沿用上一题得到的用户名 `JonahExplorer` 搜索 GitHub，可以定位账号及公开仓库 `adventurecodes`。仓库当前 README 表面上写着想去中国长城，并提到航班 `UA5040`；直接提交 `china` 会失败，说明不能只看工作树当前内容。

题目特别强调 push/pull 和 tree，应继续检查所有引用与提交历史：

```bash
git clone https://github.com/JonahExplorer/adventurecodes.git
cd adventurecodes
git branch -a
git log --all --oneline --decorate
git log --all -p -- README.md
```

提交 `3dacc60a4c45c152ee88d86684276b41ade9aa7a` 曾向 README 增加一条更正：Jonah 表示自己并不了解这些操作，下一站不是中国，而是 Italy；他会在离开 West Loop 的酒店后前往那里。该修改可在 [GitHub 的不可变提交页面](https://github.com/JonahExplorer/adventurecodes/commit/3dacc60a4c45c152ee88d86684276b41ade9aa7a) 核对，即使默认分支后来没有显示这行内容，提交对象仍保留证据。

因此 flag 为：

```text
uiuctf{italy}
```

## 方法总结

Git 仓库的当前文件只是一份快照，分支、历史提交和已删除内容都可能保存早先的信息。OSINT 调查 GitHub 时应执行 `git log --all` 并查看 diff，而不是只浏览默认分支。使用完整 commit SHA 记录证据，也比依赖可能变化的分支名更稳定。
