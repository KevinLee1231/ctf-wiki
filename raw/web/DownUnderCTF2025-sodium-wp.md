# sodium

## 题目简述

同一个 Pound 反向代理按 `Host` 分发三个站点：公开 Nginx、客户域名扫描 Flask 站，以及 h11 实现的内部 RPC 服务。取得 flag 需要串联三处缺陷：客户站用有漏洞的 URL scheme 黑名单读取本地 `.env`；Pound 4.15 与 h11 0.15.0 对 chunked 请求边界的理解不同，可走私内部 `/update_allowlist` 请求；RPC 的日志被二次 Python format 处理，可由带花括号的错误日志读取进程全局环境。

RPC 只有当 `X-Forwarded-For` 在 allowlist 中且 `Auth-Key` 正确时才进入 `/stats`。因此任何一步单独成功都不足以取 flag：先泄露 key，后建立允许来源，最后才能触发日志格式化链。

## 解题过程

### 从客户扫描器读取 RPC key

客户站禁止 `file`、`gopher`、`dict`，但只取 `urlparse(url).scheme`：

```python
BLACKLIST = ['file', 'gopher', 'dict']

def is_safe_url(url):
    return urlparse(url).scheme not in BLACKLIST

preview = urlopen(url).read().decode('utf-8', errors='ignore')
```

Python 3.11.3 受 CVE-2023-24329 影响：在 URL 前添加空格时，`urlparse` 看不到 `file` scheme，而 `urlopen` 仍会读取该 URL。注册并登录普通客户账号后，提交：

```text
 file:///proc/self/cwd/.env
```

结果页会回显文件内容，得到 `AUTHENTICATION_KEY`。这是两个进程分别持有同一 key 的配置错误；不要把竞赛实例具体 key 写死，提取到运行时变量即可。

### 走私 allowlist 更新请求

Pound 配置把 `dev.rpc-service.ductf` 转发到 h11 服务；Dockerfile 固定了 Pound 4.15 和 h11 0.15.0。h11 0.15.0 接受 chunk data 后任意两个字节作为 chunk 终止符，而 Pound 对 oversized/chunked 请求的边界处理不同。构造一个保持连接的外层 chunked 请求，在一个长度为 5 的 chunk 后放 `AAAAAXX`，让两端对 `XX` 的解释分叉；之后的字节被 h11 当作下一请求。

走私进去的第二个请求使用刚泄露的 key，伪造初始允许地址并更新 allowlist：

```http
GET /update_allowlist?ip=<proxy-observed-ip> HTTP/1.1
Host: dev.rpc-service.ductf
Auth-Key: <AUTHENTICATION_KEY>
X-Forwarded-For: 127.0.0.1
Transfer-Encoding: chunked

0

```

`<proxy-observed-ip>` 不能预先假定。先向 RPC 正常发一次未授权请求；403 正文会回显服务端实际收到的来源地址。部署可能有多个代理实例，故官方解题脚本会收集所有观察到的地址，并对每个地址走私一次。之后，带该地址的正常请求便能通过 allowlist 和 `Auth-Key` 检查。

### 污染日志并触发二次 format

`/stats` 自己解析 query string，每个参数都假定有 `=`：

```python
for param in re.search(r'.*\?(.*)', target).groups()[0].split('&'):
    params[param.split('=')[0]] = bool(param.split('=')[1])
```

提交一个没有等号的参数会抛 `IndexError`，而异常日志包含完整 `target`。将参数写成以下格式字段，即可把它持久化在 `debug.log`：

```text
/stats?{config.__init__.__globals__[os].environ[FLAG]}
```

随后以正确 key 请求 `/stats?get_log=true`。`get_logs()` 将日志拼到模板中，第一次 `.format(logs=...)` 不会解释被插入日志内部的花括号；函数结尾却再次执行 `template.format(config=config)`：

```python
template = '<h1>Admin Stats Page</h1>{logs}'.format(logs=get_logs())
template += '<h2>Current Configuration</h2>{config}'
return template.format(config=config)
```

第二次 format 解析日志中的字段。`config.__init__.__globals__` 可访问 `os` 模块，再读 `environ['FLAG']`，所以响应中出现 flag。

### 验证

完整利用应依次观察到：客户页回显 RPC key、走私后 allowlist 包含观察到的来源、第一次 `/stats` 返回 500、第二次带 `get_log=true` 的响应含 flag。源码给出的结果为：

```text
DUCTF{th3y_s33_m3_smuggl1ng_4nd_ch41n1ng}
```

## 方法总结

- 核心技巧：scheme 解析差异泄露内部 key，HTTP 请求走私升级内部访问，再借日志的二次格式化读取全局环境。
- 识别信号：代理与后端使用不同 HTTP parser、服务以 `X-Forwarded-For` 当身份、日志内容又被当 format string 时，应按顺序检查 parser differential、授权绕过和二次解释。
- 复用要点：URL 安全校验应采用 allowlist 并在实际 fetch 前规范化；代理和后端必须使用一致、已修复的 HTTP 解析器；绝不可让日志或任意外部输入再次进入 `.format()`。
