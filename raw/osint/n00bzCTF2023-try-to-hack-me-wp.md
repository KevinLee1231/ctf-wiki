# Try to Hack Me

## 题目简述

题目给出用户名 `brayannoob` 和 TryHackMe 语义提示。目标是从 GitHub 的历史提交中找到其另一个平台用户名，再访问公开 TryHackMe 资料。

## 解题过程

搜索 `brayannoob` 可定位到对应 GitHub 账号。检查其置顶仓库的提交历史，而不是只看当前文件；某次历史提交泄露了用户名 `brayan234`。

将该用户名用于 TryHackMe 公开个人资料路径，页面中可看到：

```text
n00bz{y0u_p4ss3d_th3_ch4ll3ng3_c0ngr4tul4t10ns_7c48179d2b7547938409152641cf8e}
```

## 方法总结

公开仓库删除当前文件并不会删除 Git 历史。题名提供了目标平台，GitHub commit 则提供跨平台标识；两者结合后再用资料内容确认身份。
