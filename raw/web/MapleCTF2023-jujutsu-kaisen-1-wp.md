# jujutsu-kaisen-1

## 题目简述

部署由外层 Nginx、Varnish、应用后端和仅内网开放的 GraphQL 数据库组成。预期设计让普通请求只能到应用，而 Varnish 在 Host 为 `jjk_db` 时才转发到数据库，并把来自内部 Nginx 地址的流量视为可信。第一版配置没有固定转发 Host，外部请求可以直接控制这项路由依据。

## 解题过程

Nginx 使用 `proxy_set_header Host $host`，所以客户端给出的 Host 会被原样传到 Varnish。Varnish 看到的网络来源仍是内部 Nginx，满足其来源信任条件；若同时把外部请求的 Host 设置为 `jjk_db`，后端选择逻辑便把请求发送到内部 GraphQL 服务。

因此无需走上传、ESI 或浏览器侧信道，直接发送类似请求：

```http
POST / HTTP/1.1
Host: jjk_db
Content-Type: application/json

{"query":"{ getCharacters { name notes } }"}
```

根据实际 schema 也可先做最小字段查询，再请求角色备注。数据库初始化数据中，Choso 对应记录的 `notes` 字段包含：

```text
maple{jujutsukaisenilikeitforthehotcharactersimeantheplot}
```

## 方法总结

反向代理链中，来源 IP 和 Host 都不能在未经重写时共同充当内部身份。Varnish 确实只收到 Nginx 的内部连接，但请求头仍受最外层客户端控制，于是“可信来源 + 内部 Host”条件可由外部拼出。修复应在边界代理显式设置固定上游 Host，并让内部服务使用独立监听接口或强认证，而不是只依赖可伪造的头字段。
