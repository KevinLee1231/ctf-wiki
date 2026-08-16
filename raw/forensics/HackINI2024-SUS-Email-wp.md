# SUS Email

## 题目简述

附件是原始 `.eml` 邮件。正文和 JPEG 附件主要起干扰作用，真正的线索位于两个非标准邮件头：`Security-Token` 是 JWT，`Secret-Auth-Code` 是后续受保护文本的密码。需要解析 MIME 头、解码 JWT payload，并关联两个字段。

## 解题过程

### 提取自定义邮件头

直接查看原始邮件或用 Python 标准库解析：

```python
from email import policy
from email.parser import BytesParser

with open("email.eml", "rb") as file:
    message = BytesParser(policy=policy.default).parse(file)

token = message["Security-Token"]
password = message["Secret-Auth-Code"]
print(token)
print(password)
```

得到密码：

```text
28112023
```

### 解码 JWT payload

这里只把 JWT 当作证据容器读取，不需要伪造或验证签名。取中间段做 Base64URL 解码：

```python
import base64
import json

payload = token.split(".")[1]
payload += "=" * (-len(payload) % 4)
data = json.loads(base64.urlsafe_b64decode(payload))
print(data)
```

解码结果为：

```json
{
  "sub": "1234567890",
  "flag": "https://pastebin.com/HuQnn4r6",
  "iat": 1516239022
}
```

JWT 中的 `flag` 字段指向受密码保护的 Pastebin，使用 `Secret-Auth-Code` 的值 `28112023` 解锁即可。该外部页面的关键内容是：

```text
shellmates{ceRtifIEd_eMAIL_4N4lY$1S_pRaCTiTi0N3r}
```

正文已保留链接标识、密码来源和最终内容，因此不要求未来读者依赖临时 Pastebin 仍可访问。

## 方法总结

- 核心技巧：解析原始邮件头，将 JWT payload 中的目标位置与另一个自定义头中的密码关联起来。
- 识别信号：常规鉴权头之外出现命名突兀的 `Security-Token` 和 `Secret-Auth-Code`，应优先检查而非只分析可见正文和附件。
- 复用要点：JWT 三段可离线解码，但“能读取 payload”不等于签名有效；取证时应区分内容提取与身份验证。
