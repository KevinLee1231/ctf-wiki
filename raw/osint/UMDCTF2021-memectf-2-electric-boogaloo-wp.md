# MemeCTF 2: Electric Boogaloo

## 题目简述

题目给出一张特定 meme，要求从公开代码托管平台追踪其来源。搜索结果指向名为 `Hmmmmmmm` 的 GitHub 仓库，flag 已从当前工作树删除，但仍保留在 Git 对象历史中。

## 解题过程

对图片做精确哈希或反向图片搜索，并结合题目关键词，可定位公开仓库。克隆后先检查普通提交历史：

```bash
git log --all --oneline --decorate
git log -p --all
```

仓库结构中还包含嵌套或裸露的 `.git` 数据，因此需要枚举所有引用和不可达对象：

```bash
git reflog --all
git fsck --full --no-reflogs --unreachable
git show <candidate-object-or-commit>
```

在被删除的旧版本中可读到：

```text
UMDCTF-{f0r_th3_m3m3}
```

## 方法总结

Git 删除只改变可见版本，不会立即销毁历史对象。OSINT 定位仓库后，取证重点转为 refs、reflog、提交差异和 unreachable objects。题目仓库本身的 `Hmmmmmmm` 目录只是同一线索的本地副本，不应另写成重复 WP。
