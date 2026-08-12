# Xcaptcha

## 题目简述

题目是一个限时 Web 验证码：访问 `/xcaptcha` 后，页面展示三道 128 位整数加法，并在 1 秒后自动提交。后端把生成时间和六个运算数写入已签名的 Flask session；POST 时要求三项答案正确，且服务器记录的耗时不超过 $10^9$ 纳秒。禁用前端 JavaScript 只能阻止自动点击，无法绕过后端时间检查，因此需要保存同一会话并用程序快速完成 GET、计算和 POST。

## 解题过程

### 确认服务端校验

GET 请求的核心逻辑可以概括为：

```python
challenges = []
for _ in range(3):
    a = int.from_bytes(os.urandom(16), byteorder="big")
    b = int.from_bytes(os.urandom(16), byteorder="big")
    challenges.append((a, b))

text = str(time.time_ns())
for a, b in challenges:
    text += f",{a},{b}"
session["text"] = text
```

POST 时，服务端从同一个 session 取回时间和运算数，检查：

```python
if now - past > 1_000_000_000:
    fail("超过 1 秒限制")

if submitted != expected:
    fail("计算结果错误")
```

因此必须复用 Cookie，不能分别发起互不关联的 GET 和 POST。

### 直接模拟 HTTP 请求

页面 HTML 的三个 `<label>` 中直接包含加法表达式，表单字段依次为 `captcha1`、`captcha2`、`captcha3`。下面的完整脚本用正则提取纯数字表达式并立即提交；`BASE_URL` 和 `TOKEN` 需要替换为当前题目实例的值：

```python
import re

import requests

BASE_URL = "https://challenge.example"
TOKEN = "replace-with-your-token"

session = requests.Session()
session.get(BASE_URL + "/", params={"token": TOKEN}, timeout=3)

page = session.get(BASE_URL + "/xcaptcha", timeout=3)
page.raise_for_status()

pairs = re.findall(r"(\d+)\+(\d+) 的结果是", page.text)
assert len(pairs) == 3, f"unexpected challenge count: {len(pairs)}"

answers = [int(a) + int(b) for a, b in pairs]
payload = {
    "captcha1": str(answers[0]),
    "captcha2": str(answers[1]),
    "captcha3": str(answers[2]),
}

result = session.post(BASE_URL + "/xcaptcha", data=payload, timeout=3)
result.raise_for_status()
print(result.text)
```

Python 整数没有 JavaScript `Number` 的 53 位精度限制，因此可以直接计算这些大整数。请求成功时响应页面包含 flag。

也可以用 Selenium 读取三个标签、填充表单并在一秒内点击，但直接复现 HTTP 流程更短，也更清楚地体现了真正的会话和时限约束。

## 方法总结

- 核心技巧：使用持久 HTTP 会话，在服务端时限内自动解析并提交大整数运算结果。
- 识别信号：验证码数据出现在 HTML 中、字段名固定、前端只是定时提交，而服务端仍按 session 时间戳校验。
- 复用要点：先区分前端倒计时和后端时间限制；自动化请求时必须保存 Cookie，并避免使用会丢失大整数精度的 JavaScript `Number`。
