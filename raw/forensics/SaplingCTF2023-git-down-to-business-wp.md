# Git Down to Business

## 题目简述

附件是一个看似已清理过历史的 Git 仓库。当前分支和普通 log 中没有秘密，但旧提交被 reset 后，其对象仍留在 .git/objects 中。目标是枚举不可达对象并读取被遗弃的 blob。

## 解题过程

进入仓库后先查看常规历史和 reflog：

~~~bash
git log --all --oneline --decorate
git reflog --all
~~~

即使 reflog 不完整，git fsck 仍能列出没有任何引用指向的对象：

~~~bash
git fsck --full --no-reflogs --unreachable
~~~

本题可发现不可达提交 7b4fddad 及 blob 6f136...。使用 git show 或 git cat-file -p 读取 blob，内容为：

~~~text
host = "xx.xx.xx.xx"
secret = "maple{n0T_s0_s3cret_keY_4ft3r_4ll}"
~~~

因此 flag 为：

~~~text
maple{n0T_s0_s3cret_keY_4ft3r_4ll}
~~~

## 方法总结

git reset、删除分支或重写当前历史不会立刻删除底层对象；在垃圾回收前，它们通常还能由 reflog 或 fsck 找回。取证时应复制整个 .git 目录并避免执行 gc，再从引用、reflog、不可达提交、tree 到 blob 建立恢复链。
