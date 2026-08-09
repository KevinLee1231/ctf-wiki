# Art Gallery

## 题目简述

公开的 Node.js 画廊只暴露 HTTP 端口，内部另有 FTP 服务器和 Redis。图片列表保存在 Redis 的 `image_<art_token>` 键中，并由危险的 `node-serialize.unserialize()` 读取。文件上传固定追加 `.png`，却不检查内容；`/query` 还会执行 `curl -k -L https://<host>:<port>/`。完整利用需要把 HTTPS SSRF、FTP 主动模式、Redis 协议和不安全反序列化串起来。

## 解题过程

先登录取得 session 中的 `art_token`。准备一个 `node-serialize` 立即执行函数，例如让 Node 读取 `flag.txt` 并外带；把它包装成写入当前键的 Redis 命令：

```text
SET image_<art_token> <node-serialize-payload>
```

将这段 RESP/内联协议文本加足够空白填充后，通过图片接口上传。服务器会把任意内容保存成随机名称的 `.png`，记下该文件名。

直接让 `/query` 访问 FTP 不行，因为 curl 强制使用 HTTPS。利用 TLS session ticket poisoning：攻击者的 TLS 服务把可控 FTP 命令放进 session ticket，再通过重定向让 curl 把恢复 ticket 的 TLS 字节发到内网 FTP 端口 8021。libcurl 会缓存 DNS 60 秒，普通 DNS rebinding 会被 5 秒超时阻断；可让域名一次返回“攻击者 IP + 127.0.0.1”，在重定向时关闭攻击者端口，迫使 curl 尝试本地地址。

ticket 中的有效文本为：

```text
USER x
PASS x
PORT 127,0,0,1,24,235
RETR <uploaded-random-name>.png
QUIT
```

`24*256+235=6379`，所以 FTP 主动模式会把数据连接建立到本机 Redis。`RETR` 随后把上传的“图片”内容原样送给 Redis，覆盖 `image_<art_token>`。重新访问画廊时，中间件执行：

```javascript
serialize.unserialize(await redisClient.get(`image_${req.session.art_token}`))
```

恶意函数立即运行，得到：

```text
maple{M4N_I_L0V3_SSRFz_1N_My_SSRF5_In_my_556Fs}
```

## 方法总结

这条链的每一环都只提供有限能力：任意内容上传提供协议载荷，TLS poisoning 把 HTTPS 客户端变成对 FTP 的字节发送器，FTP 主动模式再把文件转发到 Redis，最终 Redis 键污染触发反序列化。修复不能只堵一处：应移除 `node-serialize`、校验文件内容、禁止用户控制出站目的地，并把 FTP/Redis 放在无跨服务访问的网络边界中。
