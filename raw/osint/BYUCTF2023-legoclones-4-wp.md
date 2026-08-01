# BYUCTF 2023 - Legoclones 4

## 题目简述

旧版 Wikia 个人资料曾显示“当前职业”。Legoclones 先后填写过三种内容，题目接受其中任意一种精确文本；现站已经不再展示该字段。

## 解题过程

使用 Wayback Machine 查看不同时期的用户页/留言墙快照，可恢复三次历史值：

- [2015 快照](https://web.archive.org/web/20151103190612/http://clonetrooper.wikia.com/wiki/Message_Wall:Legoclones)：`AFJROTC Cadet`；
- [2013 快照](https://web.archive.org/web/20130524050235/http://clonetrooper.wikia.com/wiki/Message_Wall:Legoclones)：`Editing Clone Trooper Wiki (my website)`；
- [2012 快照](https://web.archive.org/web/20120707154640/http://clonewars.wikia.com:80/wiki/User:Legoclones)：`Editing this wiki`。

可提交例如：

```text
byuctf{AFJROTC Cadet}
```

比赛还接受去掉括号后缀的 `Editing Clone Trooper Wiki`。

## 方法总结

消失的个人资料字段通常只能靠网页归档恢复。要比较多个时间点，并把字段文本原样抄录；大小写、括号和空格都可能影响提交。
