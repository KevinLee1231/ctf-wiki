# FBI Open The Door!! 6

## 题目简述

本题继续分析 `C:\Windows\Temp\gophish\gophish.db`，要求恢复嫌疑人登录 GoPhish 管理页面时使用的明文密码。

[原始 E01 检材（百度网盘，提取码 `2vmz`）](https://pan.baidu.com/s/1oo6k9svcSJHaXo-9R5hDJg?pwd=2vmz)

## 解题过程

GoPhish 管理员账户位于 SQLite 的 `users` 表。先查询账号和完整哈希：

```bash
sqlite3 -header -column gophish.db \
  "SELECT username, hash FROM users;"
```

哈希以 bcrypt 标识开头。图形化数据库工具的单元格宽度可能只显示尾部，破解时必须复制完整字段。可以直接导出为 John the Ripper 支持的 `用户名:哈希` 格式：

```bash
sqlite3 gophish.db \
  "SELECT username || ':' || hash FROM users;" > users.hash

john --format=bcrypt \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  users.hash

john --show --format=bcrypt users.hash
```

字典命中管理员 `admin` 的密码：

```text
qwertyuiop
```

最终提交：

```text
0xGame{qwertyuiop}
```

## 方法总结

`users.hash` 是 bcrypt，不能像 NTLM 那样做普通反查，但弱口令仍会被字典攻击快速恢复。取证时应直接从 SQLite 导出完整字段，先根据哈希前缀选择正确格式，再记录字典、工具参数和最终账号对应关系。
