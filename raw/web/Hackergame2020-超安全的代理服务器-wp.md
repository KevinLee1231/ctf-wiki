# Hackergame2020 超安全的代理服务器 WP

## 题目简述

题目分两步：先从 HTTP/2 Server Push 中取得短时有效的代理 Secret 和第一段 flag；再利用 CONNECT 代理对 IPv6 回环地址过滤不完整，访问服务端本机的管理中心。

## 解题过程

### 获取 HTTP/2 推送资源

首页正文提示服务器已经“PUSH”了 Secret。源码调用 `http.Pusher.Push(secureURL, options)`，随机路径中的页面包含 Secret 与第一段 flag。普通开发者工具可能不列出该推送资源，可以用支持 HTTP/2 调试的 `nghttp`：

```shell
nghttp -sny https://target/
```

统计输出中带 `*` 的请求是服务器主动推送的路径。访问该路径即可取得 60 秒有效的 Secret，以及：

```text
flag{d0_n0t_push_me}
```

也可以导出浏览器 TLS 会话密钥后在 Wireshark 中解密 HTTP/2 流量，但命令行方案更直接；抓包截图只展示可文本化的帧字段，因此不归档。

### 绕过代理地址过滤

代理要求 `CONNECT` 请求带 `Secret` 头。源码先拒绝全局单播地址，再对 `To4()` 成功的地址过滤 `127/8`、`10/8`、`172/8` 和 `192.168/16`。IPv6 回环 `::1` 既不是全局单播，也不会进入 IPv4 分支，所以被错误放行。

使用代理访问 `[::1]:8080`，并满足管理中心对来源的检查：

```shell
curl 'http://[::1]:8080/' \
  --proxy 'https://target:443' \
  --proxy-insecure \
  --proxy-header 'Secret: 当前推送的值' \
  -H 'Referer: https://target'
```

CONNECT 隧道建立后，请求落到服务端本机管理中心，返回第二段：

```text
flag{c0me_1n_t4_my_h0use}
```

## 方法总结

HTTP/2 Push 是独立响应，不等于首页正文；排查时应看协议级请求流。代理的 SSRF/内网访问防护必须统一规范化 IPv4、IPv6、IPv4-mapped IPv6、域名解析结果及重绑定场景，不能只堆 IPv4 黑名单。允许列表通常比手写私网黑名单可靠。
