# Cosmos 的新语言

## 题目简述

页面生成一个短时有效的 `token`，并在 `/mycode` 中动态给出一串变换操作。服务端对 token 按这串操作编码后再比较提交值，且规则每 5 秒刷新一次。解题关键是读取当前规则，按逆操作还原，并在同一时间窗口内提交。

## 解题过程

源码对 `mycode` 执行 `eval`，其中可能出现以下操作：

```php
base64_encode($token);
encrypt($token);       // 每个字符的字节值加 1
strrev($token);
str_rot13($token);
```

对应的逆操作分别是 Base64 解码、每字节减一、反转字符串和 ROT13。脚本应先请求页面取得 token，再立即读取 `/mycode` 并提取操作序列。官方题解中的规则是按页面列出的顺序依次执行对应逆操作；实际复现时应以源码中表达式的嵌套方向为准，可先用一组已知字符串确认顺序。

```python
import base64
import codecs
import requests

session = requests.Session()
page = session.get("http://target/").text
token = extract_token(page)       # 按页面结构提取
ops = extract_ops(session.get("http://target/mycode").text)

value = token
for op in ops:
    if op == "base64_encode":
        value = base64.b64decode(value)
    elif op == "encrypt":
        value = bytes((b - 1) & 0xff for b in value)
    elif op == "strrev":
        value = value[::-1]
    elif op == "str_rot13":
        value = codecs.decode(value.decode(), "rot_13").encode()

print(session.post("http://target/", data={"token": value}).text)
```

若抽取出的 token 初始为文本，应在进入字节运算前统一编码；提交时再按服务端接收格式转换。整个“取 token—取规则—逆变换—提交”的过程必须在 5 秒内完成。

## 方法总结

- 核心技巧：把动态生成的变换序列视为一段可解析的指令流，为每种操作建立逆函数。
- 识别信号：服务端同时公开输入、变换规则和输出校验时，重点通常不是猜 token，而是自动逆解释规则。
- 复用要点：复合函数真正的逆序取决于源码如何组合表达式；时间窗口很短时应复用同一 HTTP 会话并避免人工复制。
