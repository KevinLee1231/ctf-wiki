# Gitastic 4

## 题目简述

当前版本中曾经存在的 `apikey.txt` 已被删除。flag 保存在删除前的文件内容中，需要定位删除提交并读取其父提交，而不是尝试普通文件恢复。

## 解题过程

先只筛选删除文件的历史：

    git log --all --diff-filter=D --summary

结果显示提交 `f3361...` 删除了 `apikey.txt`。删除提交本身已经没有该路径，因此要从它的第一父提交取文件：

    git show f3361^:apikey.txt

输出为：

    byuctf{But_th3s_was_d3l3t3d?}

也可以用 `git log --all -- apikey.txt` 确认该路径的演变，但读取时仍要明确选择删除前的树对象。

## 方法总结

Git 的“删除”只会让新树不再引用该文件，旧提交仍保留内容。标准恢复流程是定位删除提交、选择父提交、再用 `commit:path` 读取 blob。只有经过垃圾回收且对象不可达时，才需要进一步检查 reflog 或悬空对象。
