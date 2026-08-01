# BYUCTF 2022 - Alpine 3

## 题目简述

本题继续使用 Alpine 1、2 的同一磁盘证据，要求以 `mmm dd` 格式给出攻击发生日期。前两题已经确定持久化文件是 `authorized_keys`，因此可围绕该文件建立时间线。

## 解题过程

原始镜像未包含在当前 Git 仓库中，证据入口仍是 Alpine 1 题面中的 [Box 附件](https://app.box.com/s/mi71hnua1osbnaludkxvmnbj1p65bi66/file/951434445088)。进入系统后查看 SSH 目录的详细时间：

```bash
ls -al /home/mjohnson/.ssh/
```

`authorized_keys` 的修改时间显示为 `Apr 28`。结合 Alpine 1 中命令历史所揭示的写入行为，可以把这个时间解释为攻击者安装公钥持久化的日期，而不只是任意一次读取时间。

按题目要求使用英文月份缩写的小写形式提交：

```text
byuctf{apr 28}
```

## 方法总结

时间线应围绕已经确认的恶意行为建立。这里使用的是文件修改时间，因为攻击动作正是写入 `authorized_keys`；若镜像文件系统提供更完整的时间戳，还应区分修改、状态变更和访问时间，避免把一次后续查看误当成入侵时间。
