# Migurimental

## 题目简述

题目由 nginx 后的两个 Next.js 16.2.9 应用组成，flag 被分成两半：

- `backstage1`：只有 Miku（数据库中 `id=1` 的 VIP）能进入 backroom；
- `backstage2`：根页面 middleware 要求内部头 `x-real-migu: 1.3.3.7`。

仓库中的 `solution/solve.py` 是空文件，因此没有官方求解实现可依赖。下面的链条以仓库源码为主，并与[固定提交版本的参赛题解](https://github.com/Abdelkad3r/SekaiCTF-2026/blob/ef87965f07e32162595457285da138f181492f54/web/migurimental/README.md)给出的实测请求相互核对。

## 解题过程

### 1. 泄露 Miku 的 ticket

先注册普通账号，取得自己的 `session`、`ticket_uuid` 和用户 ID。访问卡页面用 `query.id` 查询任意用户并把 ticket UUID 编码进 QR；middleware 则检查查询参数 ID 是否等于 JWT 的 `sub`。

构造：

```text
/access-card?nxtPid=<our_id>&id=1
```

`nxtP` 是 Next.js 内部查询前缀。middleware 规范化后看到的是自己的 ID，允许请求；页面的 `getServerSideProps` 仍取得真实 `id=1`，于是返回 Miku 的 access card。解码 QR 得到 Miku 的 ticket UUID。

这里泄露的不是 Miku 密码或 session。数据库启动时把 `id=1` 固定为 `miku/VIP`，但密码哈希和 ticket 每次启动都由随机 UUID 生成；真正需要的是页面嵌入 QR data URL 的当前 ticket。

### 2. 重复 Cookie 解析差异

`/backroom` 有两层检查：

1. middleware 要求 cookie ticket 等于 JWT 中自己的 ticket；
2. SSR 页面要求 cookie ticket 在数据库中属于 `id=1`。

发送重复 cookie：

```http
Cookie: session=<our_session>; ticket_uuid=<miku_ticket>; ticket_uuid=<our_ticket>
```

middleware 与 Node SSR 对重复同名 cookie 取值顺序不同：middleware 看到自己的 ticket，页面看到 Miku 的 ticket，因此返回第一半 flag。

具体而言，Edge middleware 的 `request.cookies.get("ticket_uuid")` 取后出现的普通用户 ticket，与 JWT 内的 `ticketUuid` 一致；Node 页面侧的 `req.cookies.ticket_uuid` 取先出现的 Miku ticket，再由数据库查询确认其用户 ID 为 1。第一半为：

```text
SEKAI{7h3_l33k_15_b4ck_7h3_cr0wd_15_ch33r1ng_4nd_7h3_
```

### 3. `assetPrefix` 绕过第二个 middleware

`backstage2` 的 middleware matcher 只保护 `/`，而配置设置：

```javascript
assetPrefix: '/cdn'
```

从公开 `/rejected` 页面读取 Next build ID，然后请求：

```text
/cdn/_next/data/<build-id>/index.json
```

原始路径不匹配 `/`，middleware 被跳过；随后 asset-prefix 内部 rewrite 把它映射到根页面的数据路由，执行 `getServerSideProps()` 并返回第二半 flag。

nginx 会强制以 `$remote_addr` 覆盖 `X-Real-Migu`，所以伪造请求头无效；漏洞发生在 Next.js 自身的处理次序：middleware matcher 先检查外部路径，`assetPrefix` rewrite 后执行页面数据路由，却不再回到根路径 middleware。第二半为：

```text
c0nc3r7_c4n_f1n4lly_b3g1n_m1ku_m1ku_b34mmmmmmmmmmmm}
```

拼接两半得到：

```text
SEKAI{7h3_l33k_15_b4ck_7h3_cr0wd_15_ch33r1ng_4nd_7h3_c0nc3r7_c4n_f1n4lly_b3g1n_m1ku_m1ku_b34mmmmmmmmmmmm}
```

## 方法总结

三个漏洞都来自同一原则：授权层和业务层看到的请求并不相同。内部查询参数规范化、重复 Cookie 解析和 asset-prefix rewrite 都发生在不同阶段。敏感数据应在 `getServerSideProps` 的最终读取点重新授权；同时应拒绝 `nxtP*` 内部参数、重复的安全 Cookie，并把根页面对应的 `/_next/data/<build>/index.json` 及所有前缀 rewrite 一并纳入保护范围。
