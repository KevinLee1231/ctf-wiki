# UIUCTF 2023 Finding Artifacts 2 Writeup

## 题目简述

题目要求找出纽约地铁开工时“第一铲”如今收藏于哪家博物馆，并提示目标物以浅蓝色装饰闻名。flag 格式为博物馆名称的小写下划线形式。

## 解题过程

先组合事件、器物和城市进行检索：

```text
New York subway groundbreaking first shovel museum
```

检索结果指向 1900 年 3 月 24 日在 City Hall 台阶前举行的地铁开工仪式。纽约市长 Robert Van Wyck 当时使用一把 Tiffany & Co. 制作的银质礼仪铲破土，标志着 Interborough Rapid Transit 地铁工程正式开工。

继续核对博物馆自己的展品说明，可以确认：

- 该物为银与木制成的礼仪铲；
- 用途是 1900 年纽约第一条地铁的破土仪式；
- 后来还在 1979 年地铁通车 75 周年纪念活动中再次使用；
- 它由 Museum of the City of New York 收藏，并曾陈列于 `New York at Its Core` 展览。

这些信息可由 [Museum of the City of New York 的地铁纪念展品说明](https://www.mcny.org/story/contemplating-and-commemorating-rapid-transit-new-york-city) 交叉确认；链接只作为原始证据入口，关键事实已在上文完整列出。

因此答案为：

```text
uiuctf{museum_of_the_city_of_new_york}
```

## 方法总结

历史器物定位应把事件日期、人物、制造者、材质和现藏机构串成证据链。本题中“第一把铲”不是泛指施工工具，而是 Tiffany 制作的礼仪展品；锁定 1900 年 City Hall 破土仪式后，再用博物馆官方展品说明确认收藏机构，能够排除只报道事件却未持有实物的资料站点。
