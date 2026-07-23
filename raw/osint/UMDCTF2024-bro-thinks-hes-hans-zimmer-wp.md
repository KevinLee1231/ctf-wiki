# bro thinks hes hans zimmer

## 题目简述

题目给出一张 Google Street View 的 $360^\circ$ 居民区全景，并说明目标是一名在该地点活动的音乐人。需要结合街景地域特征和公开音乐人资料找出其艺名，提交格式为 `UMDCTF{musician_name}`。

![Baton Rouge 街区中大型阔叶树、独栋住宅和宽阔草坪的全景](UMDCTF2024-bro-thinks-hes-hans-zimmer-wp/baton-rouge-neighborhood.jpg)

## 解题过程

先把题目中的 Dune 叙事当作检索提示，而不是直接猜 Hans Zimmer。全景中的宽阔草坪、大型常绿阔叶树、独栋住宅和湿热地区植被更接近美国路易斯安那州的城市住宅区。围绕 Baton Rouge 缩小范围后，再检索当地音乐人及其公开活动页面。

公开的 ReverbNation 艺人页把 [Gom Jabbar](https://www.reverbnation.com/gomjabbarmusic) 标为来自 Baton Rouge 的 Singer Songwriter，并列出 Jazz、Hip-Hop、Reggae、Alternative Rock 等风格。这同时满足三个独立条件：

1. 活动城市与街景地域判断一致；
2. `Gom Jabbar` 是音乐人使用的名称；
3. 名称直接来自 Dune 世界观，与题面持续使用的 Dune 暗示吻合。

因此目标艺名为 `Gom Jabbar`，按下划线格式提交：

```text
UMDCTF{Gom_Jabbar}
```

## 方法总结

这类人物定位题应先从街景建立地域候选，再用职业、城市和题面主题做交叉验证。Dune 相关词只能用于缩小公开资料的检索范围，不能单独作为结论；最终答案还需要由音乐人页面中的所在地和身份信息共同支撑。
