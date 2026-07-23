# UMDCTF2022 Unaccounted For Co-Worker Writeup

## 题目简述

题目要求根据一段行程描述确定航班机型和最终城市：

- 2 月 24 日乘 American Airlines，从 “Mile High City” 飞往拥有最大历史剧院区的城市；
- 数日后再乘火车向西北行驶 7 站；
- 两个答案以下划线分隔。

比赛发生在 2022 年，因此航班历史查询应锁定 2022 年 2 月 24 日，而不是使用当前班表。

## 解题过程

“Mile High City” 是 Denver，对应丹佛国际机场 `DEN`。洛杉矶市中心的 Broadway Historic Theatre District 被称为美国国家史迹名录中最早且规模最大的历史剧院区，因此目的城市为 Los Angeles，对应 `LAX`。

在历史航班数据库中按以下条件筛选：

```text
date: 2022-02-24
airline: American Airlines
origin: DEN
destination: LAX
```

匹配航班的执飞机型为 Airbus A319，题目要求的机型部分写作：

```text
A319
```

第二段从 Los Angeles Union Station 沿 Pacific Surfliner 等共线列车向西北计站。按当时停站顺序，可依次经过 Burbank Airport、Van Nuys、Chatsworth、Simi Valley、Moorpark、Camarillo，到第 7 站 Oxnard，因此最终城市为：

```text
Oxnard
```

组合后得到：

```text
UMDCTF{A319_Oxnard}
```

## 方法总结

行程 OSINT 要先把自然语言别名解析成机场代码，再查询指定日期的历史记录；当前航线或当前机型不能替代当日证据。列车部分则要明确起点、方向和“到达第 7 站”的计数口径。航班与铁路班表都会变化，复盘时应记录查询日期和当时的停站序列。
