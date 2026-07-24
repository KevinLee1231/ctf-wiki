# UMDCTF 2018 - web4

## 题目简述

附件 `src.zip` 表面只包含一个没有敏感内容的 PHP 页面，但压缩包中完整保留了 `.git` 目录。flag 不在当前工作树，而在 Git 提交历史的元数据中，因此归入取证。

## 解题过程

解压时需要保留隐藏文件，确认目录中存在：

```text
src/.git/
src/.hidden
src/index.php
```

进入 `src` 后查看提交历史：

```bash
git log --oneline --all
```

输出为：

```text
b961e3e You're close! Keep looking =)
5a2545f UMDCTF-{i_th0ught_you_w3r3_forbidd3n}
```

进一步检查 `5a2545f` 可以确认它由作者 `lumpus <flag@ctf.com>` 于 2018-04-06 创建，提交中新增了 `index.php`。较新的 `b961e3e` 只新增空文件 `.hidden`，用“继续寻找”的提交信息把注意力引向历史记录。

所以 flag 是：

```text
UMDCTF-{i_th0ught_you_w3r3_forbidd3n}
```

无需猜测被删除的网页内容；该字符串直接存在于仓库对象数据库的提交信息中。

## 方法总结

源码压缩包中的 `.git` 目录会保留提交、树、作者和提交信息等历史证据。审计泄露源码时，不能只查看当前文件，还应检查 `git log --all`、历史差异和不可达对象；同时要保留解压时的隐藏文件。
