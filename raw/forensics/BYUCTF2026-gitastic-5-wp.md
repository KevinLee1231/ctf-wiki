# Gitastic 5

## 题目简述

最后一题利用 Git replace 引用，把某个原始对象透明替换为另一个对象。普通 `git show` 会自动应用替换，看见的是伪装后的历史；flag 位于被遮蔽的原对象中。

## 解题过程

replace 引用未必随普通分支一起取得，因此先显式获取并检查：

    git fetch origin 'refs/replace/*:refs/replace/*'
    git show-ref refs/replace

输出表明对象 `b82f...` 存在替换关系。Git 提供 `--no-replace-objects` 来禁用这层透明重定向：

    git --no-replace-objects show b82f...

此时显示原始对象内容，其中包含：

    byuctf{I_lov3_s3cr3t_files}

也可以通过 `git replace -l` 枚举被替换对象，但读取原历史时必须关闭 replace 机制。

## 方法总结

Git replace 会影响大量对象解析命令，所以“命令输出正常”并不等于看到原始对象。取证时应检查 `refs/replace/`，同时比较启用和禁用替换的结果。克隆或打包证据时，也要确认这些非标准引用是否被一并保留。
