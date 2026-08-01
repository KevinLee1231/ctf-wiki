# GlacierCTF 2025 gitresethard

## 题目简述

附件是一份裸 Git 仓库的磁盘归档。题面说明某人执行了 `git reset --hard` 并强制推送，正常分支历史中已经看不到被删内容。

`reset` 和 force-push 只移动 ref，并不会立刻删除对象数据库中的 commit、tree 和 blob；只要尚未执行垃圾回收，就可以从不可达对象恢复证据。决定性工作是恢复 Git 仓库中的历史事实，因此归入取证。

## 解题过程

### 1. 对裸仓库枚举不可达对象

裸仓库没有工作区，直接执行普通 `git status` 或 `git checkout` 会报错。进入解包后的仓库目录，或通过 `--git-dir` 明确指定对象库，然后运行：

```bash
git fsck --full --lost-found
```

输出中出现：

```text
dangling commit 6a81c76ebba614823433d7caf0ea7e523a998fcb
```

`dangling commit` 表示该提交没有任何分支或标签可达，但对应对象仍在 `.git/objects` 或 pack 中。用对象 ID 直接读取，无需先恢复分支：

```bash
git show 6a81c76ebba614823433d7caf0ea7e523a998fcb
```

提交中的 `carpet/shit` 保存了一个完整解密命令，其中包括 Base64 密文、AES 模式、PBKDF2 参数和明文密码 `tJnAQZQF2bKx4`。

### 2. 解开提交中留下的密文

密文为 OpenSSL `Salted__` 格式。可以按提交中的原命令解密；为避免某些 shell 不支持进程替换，可改成管道：

```bash
echo 'U2FsdGVkX18liMZqk4AiqSRX5HZpfrnZAmrfRaS1UztVewZqjgX1wTHCNNj2H5crA/0VUhBXMk9bo/N/lKfFPQ==' \
  | base64 -d \
  | openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:tJnAQZQF2bKx4
```

输出为：

```text
gctf{0113_wh0_g1t_r3s3t3d_th3_c4t_4789}
```

### 3. 验证证据链

恢复结果应同时满足三点：对象 ID 能被 `git cat-file -t` 识别为 commit；`git show` 展示的 tree 中确有解密脚本；解密输出与仓库中题目 flag 一致。这样能够排除“从其他文件直接抄到 flag、但并未恢复被删提交”的伪复现。

## 方法总结

Git 的 ref、reflog 和对象数据库是三个不同层次。`git reset --hard` 会改变索引和工作区，force-push 会改变远端 ref，但已写入的对象通常要到 reflog 过期并被 `git gc --prune` 后才真正消失。面对磁盘或裸仓库证据，应优先使用 `git fsck --full` 枚举不可达对象，再用 `git show`、`git cat-file` 逐层还原 commit、tree 和 blob。本题最终的密码学只是对取证所得脚本的直接复现。
