# DownUnderCTF 2021 - The Internet is Written in Ink

## 题目简述

题面说实习生 Ducky 曾把比赛数据公开到“CTF clock”，随后通过 `404` 删除，并强调互联网内容像墨水一样难以抹去。“CTF clock”指向 CTFtime，`404` 与标题则提示使用网页存档查看已删除的历史内容。

## 解题过程

先定位 DownUnderCTF 2021 的 [CTFtime 赛事页](https://ctftime.org/event/1312)，再把该 URL 放入 [Internet Archive 的历史快照列表](https://web.archive.org/web/*/http://ctftime.org/event/1312)。

2021 年 3 月 14 日的存档仍保留了后来从当前页面删除的一行文本：

```text
Ok, ok — here is your flag: a5abef5222adc680a453607384bcb4d2
```

题面要求把找到的值放入标准 flag 外壳，因此结果为：

```text
DUCTF{a5abef5222adc680a453607384bcb4d2}
```

这里的两个链接只用于追溯来源；即使未来网页或存档索引发生变化，所需的目标页面、快照日期和被删除内容已经完整保留在正文中。

## 方法总结

“删除”“404”“互联网不会遗忘”等提示通常指向 Wayback Machine 一类历史网页服务。复现时要存下原始 URL、快照日期和关键历史文本，而不是只记录存档首页链接；同一页面若有多个快照，还应比较时间线，确认内容确实在何时出现或消失。
