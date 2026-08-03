# safepy

## 题目简述

题目实现了一个“对输入表达式关于 $x$ 求导”的服务。作者因担心 `sympify` 不安全，改用了 `sympy.parsing.sympy_parser.parse_expr`，但没有传入受限的 `local_dict`、`global_dict` 或安全转换器：

```python
from sympy import *

def parse(expr):
    return parse_expr(expr)

user_input = input('Your expression: ')
expr = parse(user_input)
print(diff(expr, Symbol('x')))
```

这不是安全修复。SymPy 的[官方 `parse_expr` 文档](https://docs.sympy.org/latest/modules/parsing.html)明确说明该函数会使用 `eval`，不能处理未清洗输入；默认全局字典还会按 `from sympy import *` 初始化。攻击者可以在“解析”阶段直接执行 Python 调用，flag 文件位于 `/flag`。

## 解题过程

向服务提交：

```python
print(open('/flag').read())
```

`parse_expr` 会先把字符串转换为 Python 代码，再在包含 SymPy 名称和 Python builtins 的环境中求值。求值顺序如下：

1. `open('/flag').read()` 读取文件；
2. `print(...)` 在解析阶段立即把内容写到标准输出；
3. `print` 返回 `None`；
4. 后续 `diff(None, Symbol('x'))` 可能报错，但 flag 已经通过副作用泄露，后续异常不影响结果。

仓库的官方 healthcheck 使用的正是这一 payload，并从输出中提取：

```text
uiuctf{na1v3_0r_mal1ci0u5_chang3?}
```

如果希望保留正常求导流程，也可以让副作用执行后返回一个合法 SymPy 表达式，例如：

```python
print(open('/flag').read()) or x
```

这样解析结果为符号 $x$，后续求导得到 $1$，连接不会因 `None` 报错而提前结束。

## 方法总结

- 核心技巧：利用 `parse_expr` 内部的 `eval` 在符号表达式解析阶段执行任意 Python 调用，并通过 `print` 副作用读取 flag。
- 识别信号：应用把不受信任字符串直接传给 SymPy 字符串解析器，同时误以为“数学解析函数”天然比 `eval` 或 `sympify` 安全。
- 复用要点：更换 API 名称不等于修复代码执行。需要不受信任的数学表达式时，应自行定义可接受的 token/AST 白名单和允许函数表，并在低权限隔离进程中计算；不能把 `parse_expr` 当成安全沙箱。
