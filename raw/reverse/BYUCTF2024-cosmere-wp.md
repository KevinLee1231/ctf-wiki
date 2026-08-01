# Cosmere

## 题目简述

题目询问官方 Knights Radiant 测验中，理想 Dustbringer 在 “Free-spirited—Disciplined” 维度上的 disciplined 百分比。网站的评分逻辑完全在浏览器端运行，答案隐藏在 JavaScript 的理想特征数组中，而不是靠手工反复填写问卷猜测。

## 解题过程

打开[官方 Knights Radiant 测验](https://www.brandonsanderson.com/pages/official-knights-radiant-order-quiz)，在 BYUCTF 2024 比赛版本的前端资源中搜索 `traitData`；若线上代码已更新，应使用浏览器缓存或比赛快照复现。该三维数组按“问题、Order、理想滑块值”组织。比赛版本的页面 HTML 漏掉了内部编号第 4 题，但 JavaScript 数组仍保留该项，导致后续可见题号与数组索引错一位。

逐项对齐标签后：

```text
维度：Free-spirited -> Disciplined
内部问题编号：15
Dustbringer 的 Order 索引：2
traitData 对应值：87
```

因此理想 Dustbringer 为 87% disciplined：

```text
byuctf{87}
```

## 方法总结

客户端问卷的全部评分常随静态资源下发。逆向时要同时核对 DOM 展示顺序与数据数组索引；本题最容易出错的不是取值，而是缺失 HTML 问题造成的 off-by-one。
