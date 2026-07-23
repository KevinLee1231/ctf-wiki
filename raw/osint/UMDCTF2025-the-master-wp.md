# UMDCTF 2025 - the-master

## 题目简述

题目给出一张俄亥俄州小镇街景，要求提交拍摄位置所在道路的完整地址。

![Lore City Main Street 全景：红砖教堂、历史标志和道路布局用于确定具体路线节点](UMDCTF2025-the-master-wp/lore-city-church.jpg)

## 解题过程

原图没有可利用的 GPS 元数据。画面中的红砖教堂是醒目的建筑候选，路旁历史标志则
提供了更直接的文字线索：它属于 John Hunt Morgan Heritage Trail。这个路线在
俄亥俄州经过多个小镇，所以仅识别路线名称还不够，必须继续用教堂外观和街道布局
缩小范围。

检索该路线的地点列表，可以找到 Lore City 的“A Destructive Spree”标记。
[历史标记资料](https://www.hmdb.org/m.asp?m=171079)给出三项关键事实：

- 标记编号为 41，属于 John Hunt Morgan Heritage Trail；
- 位置在 Lore City；
- 标记位于 Main Street 与 Great Guernsey Rail-Trail 交会处，参考地址为
  `205 Main St, Lore City, OH 43755`。

回到地图核对，标记旁的砖教堂、道路弯曲方向、两侧住宅和架空线均与题图一致。
题目只要求道路而不是具体门牌，因此答案为：

```text
UMDCTF{Main St, Lore City, OH 43755}
```

## 方法总结

历史路线标牌通常能把普通街景转化为可搜索的实体，但一条路线可能包含数十个节点。
正确做法是先识别路线，再用附近独特建筑和道路关系确定具体节点，最后从地点资料读取
道路与邮编。这样结论由路线、建筑和地图三类证据共同支持。
