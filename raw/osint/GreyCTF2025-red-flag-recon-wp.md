# Red Flag Recon

## 题目简述

题目要求调查一名频繁在 GreyHats Instagram 下留言、声称要为比赛出题的用户，并寻找其公开活动中的“red flag”。核心知识点是 GitHub fork network 的历史可见性：一个私有派生仓库中的提交，在同一 fork 网络后来出现公开仓库时可能被意外暴露。

## 解题过程

从 GreyHats Instagram 评论区识别用户 `ducati777__`，再按相似用户名关联到 X 账号 `ducati777_`。该账号发过“Im writing a new challenge 🚩.”，并提到把 flag 存在 private fork 中。这既是题面双关，也是后续 GitHub 检索的方向。

继续关联到 GitHub 用户 `ducati777-o`，目标 fork 网络中的仓库为：

```text
Brainstorm-Nonsense/chal-collection
```

普通分支列表看不到已经隐藏的敏感提交，但 TruffleHog 的 GitHub 扫描会枚举仓库对象和可达的历史提交，并用秘密检测器检查内容。对目标 fork 网络运行扫描后，定位提交：

```text
385a712af8aad142f363cd2130d419ad09f68214
```

查看该提交即可取得唯一符合题目格式的字符串：

```text
grey{R4nD_n0_c4n7_h1d3_f14G}
```

该公开仓库当前已无法正常访问，但官方解答保留了账号、仓库名、工具和完整提交哈希，因此解题链仍可完整说明；这里不把已经失效的临时仓库地址保留为正文依赖。

## 方法总结

删除分支、把仓库改为私有或只在私有 fork 中提交，都不等于敏感对象已从 GitHub 的 fork network 消失。调查此类泄漏时，要从社交账号关联得到开发者身份和仓库线索，再让秘密扫描工具检查完整提交历史，而不是只搜索当前默认分支。仓库失效后，应在题解中保留可验证的关键标识和机制，不能用一个死链代替必要说明。
