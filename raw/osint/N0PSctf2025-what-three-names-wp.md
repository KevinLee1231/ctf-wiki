# What Three Names

## 题目简述

绑架案链的第二题要求找出三名间谍的姓氏，并按照他们所属 Topia 的字母顺序排列。上一题已经定位到 TopiaNews，因此本题继续调查该站点的历史内容。

## 解题过程

当前 TopiaNews 首页只保留新闻与宣传内容，直接浏览看不到绑匪对话。Wayback Machine 没有保存所需页面，但 archive.is 的旧快照中仍能找到对话入口。旧版页面使用数字会话 ID，其中 ID `1337` 的会话包含三名参与者：

```text
Cryptie
Boomer
Mondey
```

会话页展示的是昵称。继续检查快照中的用户链接和页面命名，可以定位三个资料页：

```text
crypto.html
boomer.html
money.html
```

资料页同时给出了真实姓氏和所属地区。将信息整理为：

| 所属 Topia | 会话昵称 | 姓氏 |
| --- | --- | --- |
| Cryptopia | Cryptie | Keyflipper |
| Pwntopia | Boomer | Hackpaw |
| Webtopia | Mondey | Websnare |

题目要求按 Topia 名称的字母顺序排列，因此顺序是 `Cryptopia → Pwntopia → Webtopia`，而不是按人物昵称排序。将三个姓氏转为小写并用下划线连接：

```text
N0PS{keyflipper_hackpaw_websnare}
```

## 方法总结

网站内容会变化，OSINT 调查不能只依赖当前页面。不同网页存档服务的覆盖范围也不相同：Wayback 缺失时，应检查 archive.is 等独立快照。恢复旧会话后，还要区分昵称、真实姓名和所属组织，并严格按照题目指定字段排序，避免证据正确却因拼接顺序错误而失分。
