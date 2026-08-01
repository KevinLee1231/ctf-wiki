# Gitastic 2

## 题目简述

第二题把线索放在 Git 提交作者统计中。大量普通提交掩盖了一个异常作者名，flag 就是该作者身份字段。

## 解题过程

使用 shortlog 按作者汇总全部引用中的提交：

    git shortlog -sne --all

也可以去掉邮箱只保留姓名：

    git shortlog -sn --all

列表中的异常作者名为：

    byuctf{wh0s_th3_auth0r?}

作者名是提交对象的一部分，与本机当前 `user.name` 无关；因此应读取历史元数据，不要根据工作区配置猜测。

## 方法总结

作者与提交者是两个不同字段，Git 取证时都值得检查。`shortlog` 能把重复记录聚合，使少量异常值从大量提交中突出；需要更精确时可再用 `git log --format` 导出 author 和 committer 字段交叉核对。
