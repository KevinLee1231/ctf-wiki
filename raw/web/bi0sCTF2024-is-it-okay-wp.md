# bi0sCTF 2024 - Is It Okay

## 题目简述

题目由三个 HTTP 服务组成：Flask `core`、Node.js `vec` 和只在内部网络可达的 Docker Registry。`core` 的 `/okay` 会先用 `urllib.parse.urlparse()` 检查主机名，再用 `requests.get()` 访问 URL；`vec` 则把用户提供的网卡名交给 `network.gateway_ip_for()`。

完整利用链是：用 Python 3.11.3 的 URL 前导空白解析差异绕过 `registry` 黑名单，通过 Registry API 下载 `vec` 镜像并恢复源码，在 `network` 包的命令拼接点注入 shell，最后修改部署中与 `core` 共享的 Jinja 模板，令 Flask 执行 SSTI 读取 flag。虽然仓库把题目放在 Misc，决定性漏洞均位于 Web 服务和 HTTP 协议流程，因此归入 Web。

## 解题过程

### 前导空白绕过 SSRF 主机检查

`/internal` 会列出内部服务，其中 Registry 地址为 `http://registry:5000`。`/okay` 的检查逻辑是：

```python
url = request.form.get("url")
parsed_host = urlparse(url).hostname
if parsed_host == "registry":
    return "blocked"
return requests.get(url, allow_redirects=False).content
```

镜像使用 `python:3.11.3`。该版本受 CVE-2023-24329 所涉及的 URL 前导空白解析问题影响：验证阶段对以空格开头的 URL 得不到预期的 `hostname`，而后续请求库会接受或规范化同一字符串并真正访问其中的 HTTP 地址。因此提交：

```text
 http://registry:5000/v2/
```

开头的空格让黑名单没有看到 `registry`，请求阶段却仍能连到内部 Registry。这里应直接禁用用户可控的任意 URL 请求或在规范化后重新解析、解析后再连接，并校验最终解析到的 IP；只比较原始主机字符串无法形成可靠 SSRF 防护。

### 从 Docker Registry 恢复 `vec` 源码

Registry v2 API 可以按以下顺序读取：

```text
GET /v2/_catalog
GET /v2/vec/manifests/latest
GET /v2/vec/blobs/<layer-digest>
```

所有内部请求都通过 `/okay` 代理发送，并保留 URL 开头空格。`/_catalog` 给出仓库名 `vec`；manifest 的 `fsLayers` 或新格式中的 layer digest 列出镜像层。逐层下载 blob、解压 tar，并按镜像层顺序叠加文件，注意处理删除标记，便能恢复 `/app/net.js` 和 `package.json`。

源码显示 `/custom` 直接使用查询参数：

```javascript
network.gateway_ip_for(req.query.interface, (err, out) => {
  res.send(out);
});
```

依赖为 `network@0.6.1`。该包在 Linux 上通过 shell 命令查询指定接口，接口字符串进入命令上下文却未被安全转义，所以 `interface` 中的 `;`、`|` 和注释符可以追加任意命令。

### 在 `vec` 中执行命令

先用无害命令验证，例如把 `id` 输出写入可读临时文件。通用请求形式为：

```text
/custom?interface=eth0;COMMAND;#
```

实际发送时要对分号、空格、重定向符等做 URL 编码。官方解法使用反向 shell，但这不是漏洞成立的必要条件；在受限环境中也可以用一次性 HTTP 回传或直接修改目标共享文件。关键证据是命令在 `vec` 容器身份下执行，而不是把网络超时误判成成功。

### 通过共享模板把容器 RCE 传到 `core`

官方部署让 `vec` 与 `core` 看到同一个宿主机模板目录，而 `core` 又设置：

```python
app.config["TEMPLATES_AUTO_RELOAD"] = True
```

从 `vec` 覆盖共享的 `index.html`：

```jinja2
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('cat /flag.txt').read() }}
```

再以合法会话刷新 `core` 首页，Jinja 会在 `core` 容器中执行表达式，读取只有该容器可见的 flag。

仓库快照存在一处复现差异，不能忽略：其中 `docker-compose.yml` 为 `core` 挂载 `./mnt/templates:/app/templates`，却为 `vec` 写成 `./templates:/app/templates`，而仓库中没有这个 `./templates` 目录；flag 也被挂载到 `/flag` 目录，官方短解 payload 却读取 `/flag.txt`。因此按仓库文件原样启动时，共享模板这一步未必成立。复现者应先用 `mount`、`lsblk` 或文件 inode/写入测试确认实际部署的共享路径，并按实际挂载把读取目标改为 `/flag.txt` 或 `/flag/flag.txt`。前述最后阶段描述的是官方远程部署所依赖的条件，不应把仓库 compose 的拼写差异当作已验证共享。

## 方法总结

这是一条跨服务信任链：解析器与请求库对同一 URL 的规范化不一致产生 SSRF；未鉴权 Registry 暴露了原本隐藏的应用源码；第三方网络库把接口名带入 shell；共享可写模板又把一个容器中的命令执行扩展到持有 flag 的 Flask 服务。排查此类题目时应逐层记录“当前原语位于哪个服务、能看到哪些文件、下一层信任边界是什么”，并以实际挂载验证最后的跨容器假设。
