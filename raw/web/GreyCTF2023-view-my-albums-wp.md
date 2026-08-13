# GreyCTF 2023 View My Albums

## 题目简述

应用直接反序列化 `prefs` Cookie。只要结果不是 `UserPrefs`，程序就调用 `var_dump` 后退出。现有的 `Albums::__debugInfo()` 会访问其内部 `RecordStore`，因此可构造两阶段 PHP 对象注入链：先读取数据库凭据文件，再连接数据库并查询 `flag` 表。

## 解题过程

第一阶段把 `Albums` 的私有 `store` 属性设为 `CsvRecordStore("db_creds.php")`。反序列化后对象不属于 `UserPrefs`，于是进入：

```php
echo "Unrecognized data: ";
var_dump($prefs);
```

`var_dump` 会调用 `Albums::__debugInfo()`，后者执行 `getAllAlbums()`；`CsvRecordStore` 随即用 `file()` 读取指定路径，并把每行内容作为记录返回。响应因此泄露 `db_creds.php` 中的主机名、用户名、密码和数据库名。构造 Cookie 时要按 PHP 私有属性序列化格式保留类名和空字节，并进行 URL 编码。

第二阶段把同一个 `store` 换成 `MysqlRecordStore`，填入刚获得的连接参数，并把私有 `table` 设为 `flag`。其 `__wakeup()` 会重新建立 MySQL 连接；再次触发 `Albums::__debugInfo()` 后，`getAllRecords()` 执行：

```sql
SELECT * FROM flag
```

结果随 `var_dump` 输出。部署用 `albums.sql` 中保存的实际 flag 为：

```text
grey{l4_mu5iCA_DE_haIry_FroG}
```

仓库 README 把前缀写成了 `greyctf{...}`，与建库脚本不一致；这里以实际初始化数据库的 SQL 为准。

## 方法总结

反序列化风险不只来自 `__destruct`。`var_dump` 也会触发 `__debugInfo`，而对象恢复又会调用 `__wakeup`，两者组合成了文件读取和数据库访问链。应避免对不可信 Cookie 使用原生 `unserialize`；改用带完整性保护的 JSON，并让调试输出不触发具有 I/O 副作用的魔术方法。
