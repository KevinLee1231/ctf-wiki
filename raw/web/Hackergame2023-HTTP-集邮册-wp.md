# HTTP 集邮册

## 题目简述

网页把用户输入中的字面量 `\r`、`\n` 解码为控制字符，将原始字节发送给默认配置的 `nginx:1.25.2-bookworm`，然后解析响应首行并按用户累计不同状态码。收集 5 种状态码得到第一问 flag；产生一个无法解析出状态码的非空响应得到第二问 flag；累计 12 种状态码得到第三问 flag。

状态行解析逻辑要求首个 CRLF 前能拆成三个空格分隔字段，且第二项能转为整数：

```python
crlf = buf.find(b"\r\n")
try:
    if crlf == -1:
        raise ValueError("No CRLF found")
    status_line = buf[:crlf]
    http_version, status_code, reason_phrase = status_line.split(b" ", 2)
    status_code = int(status_code)
except ValueError:
    status_code = None
```

因此主线是理解 HTTP 请求语法、条件请求、范围请求以及 nginx 对 HTTP/0.9 的兼容行为，归入 `web`。

## 解题过程

### 先收集五种常见状态码

网页输入框使用字面量 `\r\n` 表示换行。以下请求分别产生五种不同状态码：

```http
GET / HTTP/1.1\r\nHost: example.com\r\n\r\n
```

得到 `200 OK`。把路径改为不存在的 `/x` 得到 `404 Not Found`；把方法改为 `POST` 得到 `405 Not Allowed`；把版本改为 `HTTP/11` 得到 `505 HTTP Version Not Supported`；构造非法请求行：

```http
GET / aHTTP/1.1\r\nHost: example.com\r\n\r\n
```

得到 `400 Bad Request`。累计后满足第一问的五种状态码要求。

### 触发无状态码响应

只发送 HTTP/0.9 风格的请求行：

```http
GET /\r\n
```

HTTP/0.9 响应只有资源正文，没有 `HTTP-Version Status-Code Reason-Phrase` 状态行。nginx 为兼容旧客户端直接返回页面内容，网页解析器无法从首行取出状态码，因而将该响应标记为“无状态码”，满足第二问。

### 扩展到十二种状态码

在前述五种之外，可逐个发送以下请求。每次都使用新的一次提交，不要把它们拼成同一条连接上的流水线。

`100 Continue`：

```http
GET / HTTP/1.1\r\nHost: example.com\r\nExpect: 100-continue\r\n\r\n
```

`206 Partial Content` 与 `416 Range Not Satisfiable`：

```http
GET / HTTP/1.1\r\nHost: example.com\r\nRange: bytes=1-2\r\n\r\n
```

```http
GET / HTTP/1.1\r\nHost: example.com\r\nRange: bytes=114514-1919810\r\n\r\n
```

`304 Not Modified` 与 `412 Precondition Failed`：

```http
GET / HTTP/1.1\r\nHost: example.com\r\nIf-Modified-Since: Tue, 15 Aug 2023 17:03:04 GMT\r\n\r\n
```

```http
GET / HTTP/1.1\r\nHost: example.com\r\nIf-Match: "definitely-not-the-current-etag"\r\n\r\n
```

`413 Content Too Large` 不要求真的发送巨大正文，只需声明超大的长度：

```http
GET / HTTP/1.1\r\nHost: example.com\r\nContent-Length: 1145141919810\r\n\r\n
```

最后发送一个 URI 超过 nginx 请求行限制、但仍低于网页 10 MiB 总请求限制的请求，即可得到 `414 URI Too Long`。例如让路径由约 16 KiB 的 `a` 组成：

```text
GET /<重复约 16000 次的 a> HTTP/1.1\r\nHost: example.com\r\n\r\n
```

这七种加上最初五种，累计达到 12 种。还可用未知传输编码触发备用的 `501 Not Implemented`：

```http
GET / HTTP/1.1\r\nHost: example.com\r\nTransfer-Encoding: gzip\r\n\r\n
```

网页会把解析出的状态码放入每个用户对应的集合，因此重复状态码不会增加计数；以页面显示的集合达到 12 个且出现三个 flag 为验证标准。

## 方法总结

- 核心技巧：利用请求行、方法、版本、范围、条件头、长度头和传输编码触发 nginx 的不同响应分支，并用 HTTP/0.9 得到无状态行响应。
- 识别信号：服务允许发送近乎原始的 HTTP 请求，并按响应首行而非业务正文进行判定。
- 复用要点：状态码由协议状态机和资源条件共同决定；构造时应区分请求解析错误、方法不允许、条件失败和范围错误。老服务器的向后兼容路径也可能产生现代客户端少见的响应形态。
