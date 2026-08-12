# Git? Git!

## 题目简述

附件是一份完整的 Git 工作目录。题面说明有人曾把 flag 提交到本地仓库，随后撤销该提交。目标不是从当前版本搜索 flag，而是从 `.git` 中保留的引用移动记录和未被垃圾回收的历史对象恢复被撤销的提交。

附件中的 `.git/logs/HEAD` 记录了这条关键历史：

```text
15fd0a1 -> 505e1a3  commit: Trim trailing spaces
505e1a3 -> 15fd0a1  reset: moving to HEAD~
15fd0a1 -> ea49f0c  commit: Trim trailing spaces
```

`reset` 只移动分支和工作树位置，不会立刻删除提交对象，因此 `505e1a3` 仍可访问。这是一道 Git 历史证据恢复题，决定性主障碍是取证而非代码逆向。

## 解题过程

解压附件并进入含 `.git` 的仓库目录，先查看 HEAD 的本地移动历史：

```bash
git reflog --date=iso
```

输出显示 `505e1a3` 是一次提交，紧接着仓库执行了 `reset: moving to HEAD~`。不需要破坏当前工作树即可直接查看该提交中的文件：

```bash
git show --stat 505e1a3
git show 505e1a3:README.md
```

在该版本的 `README.md` 中可以找到被加入后又撤销的 flag 注释。

如果希望把整棵历史树恢复到单独位置，可使用：

```bash
git worktree add --detach ../recovered-505e1a3 505e1a3
```

这比在原目录执行 `git reset --hard` 更安全：既保留当前分支状态，也能检查被撤销提交的完整内容。验证依据是 reflog 中确有 `505e1a3 -> 15fd0a1` 的 reset 记录，且 `git show 505e1a3:README.md` 能读取对应 blob。

## 方法总结

- 核心技巧：通过 `git reflog` 找到被 reset 掉的提交，再用 `git show <commit>:<path>` 读取历史文件。
- 识别信号：题面提到“本地提交后撤销”，附件又保留完整 `.git` 目录，而当前分支看不到目标内容。
- 复用要点：`reset`、分支删除和 amend 通常只是让对象变得不可达；在 reflog 过期和 `git gc` 清理之前，提交仍可按对象 ID恢复。取证时优先只读查看或使用 detached worktree，避免覆盖现场。
