# Porg City

## 题目简述

Discord Bot 的 `porg` 命令把用户名直接拼进 SQLite 查询，并把查询结果的 `image` 字段交给 `os.path.join('/srv/images/', value)` 后作为附件发送。过滤只拒绝 `..`，因此可以用 SQL 注入伪造绝对文件路径。

## 解题过程

查询结构为：

```sql
SELECT * FROM porgs WHERE name LIKE '<name>'
```

用 `UNION SELECT` 构造与表结构相同的五列，并把 image 设置为 `/proc/self/cwd/flag.txt`：

```text
@Porg Bot porg ' UNION SELECT 1 AS id, 'hack' AS name,
1 AS age, 'e' AS fav_color, '/proc/self/cwd/flag.txt' AS image--
```

该路径不含 `..`。Python 在第二个参数为绝对路径时，会让 `os.path.join('/srv/images/', image)` 直接得到该绝对路径；`/proc/self/cwd` 又稳定指向 Bot 当前工作目录，因此无需知道容器真实目录。

比赛前 Discord 的行为会把文本文件直接显示为附件；比赛期间客户端更新后，附件可能不在 UI 中出现。此时刷新 Discord Web，打开开发者工具 Network，检查 `messages?limit=50` 响应中的 attachment/CDN URL，直接访问即可读取：

```text
byuctf{hehehe_hASWHHyrc9_https://i.imgflip.com/8l27ka.jpg}
```

仓库中的角色图和两张操作截图均为装饰或可转写界面信息，未保留为 WP 图片。

## 方法总结

利用链是 SQL 注入控制查询结果 → 绝对路径绕过错误的 join 约束 → Bot 文件发送。前端展示变化不代表后端附件消失；应沿网络响应验证 CDN 对象，而不是把客户端 UI 当成唯一证据。
