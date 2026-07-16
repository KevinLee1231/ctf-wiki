# FBI Open The Door!! 5

## 题目简述

本题沿用 `fish.E01`，要求找出嫌疑人 St5rr 在钓鱼平台中配置的 SMTP 授权码。

[原始 E01 检材（百度网盘，提取码 `2vmz`）](https://pan.baidu.com/s/1oo6k9svcSJHaXo-9R5hDJg?pwd=2vmz)

## 解题过程

在镜像的临时目录中可以发现 GoPhish 工作目录：

```text
C:\Windows\Temp\gophish
```

导出其中的 `gophish.db`。该文件是 SQLite 数据库，可以直接用 `sqlite3` 查看表结构和 SMTP 配置：

```bash
sqlite3 gophish.db ".tables"
sqlite3 -header -column gophish.db \
  "SELECT name, host, username, password FROM smtp;"
```

数据库包含 GoPhish 的活动、邮件模板、SMTP 配置和用户等表。

`smtp` 表中唯一一条记录的关键信息为：

```text
name     = St5rr
host     = smtp.qq.com:465
username = 3092158216@qq.com
password = wpdqlnyvetqyddce
```

GoPhish 为了发送钓鱼邮件而在该字段中保存了可直接使用的 SMTP 授权码。最终提交：

```text
0xGame{wpdqlnyvetqyddce}
```

## 方法总结

应用取证不能只搜索通用浏览器或系统凭据，还应定位业务程序自身的数据目录。GoPhish 默认使用 SQLite 保存配置，本题所需授权码就在 `smtp.password` 中；记录数据库路径、查询语句和同一行的账号信息，才能说明该值确实属于嫌疑人的钓鱼平台。
