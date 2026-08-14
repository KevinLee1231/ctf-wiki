# the best detective

## 题目简述

题目只提供用户名 `cingelord2361`，要求确认此人的姓名、最喜欢的歌手和最喜欢的电影，并按 `greyhats{name_favouritesinger_favouritemovie}` 组合。决定性证据来自公开账号、搜索索引和网页历史快照。

## 解题过程

先用 Sherlock 或手工搜索在多个站点枚举同名账号。命中的 [Twitter 账号](https://twitter.com/cingelord2361) 使用名为 Greyson Caterine，账号简介同时给出了她最喜欢的歌手 Miracle Johnson。

当前页面不再显示完整历史内容，因此继续在 Internet Archive 中查询该账号的历史快照。历史版本中存在一条后来删除的推文，内容引用了她最喜欢的电影；将引文反向搜索，可以确认来源是 *The Martian*。另外，以完整用户名搜索还能找到与 Will Smith、YouTube Rewind 旧梗相关的索引线索，它用于确认这些结果属于同一网络身份，而不是另一个重名账号。

按题目格式移除空格、统一为小写，并保持“姓名、歌手、电影”的顺序：

```text
greyhats{greysoncaterine_miraclejohnson_themartian}
```

## 方法总结

OSINT 结论应由多个来源相互约束：用户名负责跨站关联，个人资料给出姓名与歌手，历史快照恢复已删除内容，搜索引擎再验证引文来源。外部页面可能失效，所以 WP 已把每个来源提供的关键事实写入正文；链接仅用于复核，不是理解解法的前置条件。
