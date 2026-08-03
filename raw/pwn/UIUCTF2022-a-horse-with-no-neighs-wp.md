# A Horse with No Neighs

## 题目简述

本题修复了 `A Horse with No Names` 中正则只从字符串开头匹配的问题：`re.match(r"[a-zA-Z]{4}", ...)` 被改成 `re.search(...)`，因此输入任意位置都不能出现连续 4 个 ASCII 字母。其余机制不变：最多使用 4 类非单词字符，表达式编译后仅清空最外层代码对象的 `co_names`，再将求值结果转成列表并打乱。

修复阻止了“在开头放一个符号、后面继续写普通长标识符”的绕过，却没有解决 Unicode 标识符归一化和嵌套代码对象问题，因此仍属于 Python jail 逃逸。

## 解题过程

使用仓库提供的同一阶段一 payload：

```python
((𝘦𝘷𝘢𝘭(𝘪𝘯𝘱𝘶𝘵()) for x in ((),)))
```

它能同时满足全部限制：

1. `𝘦𝘷𝘢𝘭`、`𝘪𝘯𝘱𝘶𝘵` 由数学斜体字符构成，不匹配 ASCII 范围 `[a-zA-Z]`，但 Python 会把它们规范化为内建函数 `eval` 和 `input`。
2. 剩余 ASCII 单词只有 `for`、`in` 和单字符 `x`，不存在连续 4 个 ASCII 字母。
3. 不同的非单词字符仅为左括号、右括号、空格和逗号，共 4 类。
4. `eval(input())` 位于生成器主体。`compile` 会为该主体创建内层代码对象，而题目只把最外层 `co_names` 替换为空元组，内层名称表仍然有效。

当程序对求值结果调用 `list()` 时，生成器开始执行并读取第二行。此时输入不再经过任何正则或字节码修改，可以直接利用打印副作用：

```python
print(open('/flag.txt').read()) or ()
```

flag 在列表打乱之前就已输出：

```text
uiuctf{my_challenges_always_have_unintended_solutions_and_i_am_less_okay_with_that}
```

## 方法总结

- 核心技巧：在 `re.search` 已覆盖全文的情况下，继续利用 Unicode 规范化制造过滤器与解释器之间的语义差异，并通过生成器隐藏内层 `co_names`。
- 识别信号：补丁只把 `match` 改为 `search`，却没有改变 ASCII 字符类、编译语义或嵌套代码对象处理方式；这是表面补丁而非安全边界重建。
- 复用要点：修复绕过时必须针对漏洞根因测试，而不是只封堵已知 payload。应使用严格语法白名单或独立进程权限隔离，不能依赖 ASCII 正则和浅层 `CodeType` 修改实现 Python 沙箱。
