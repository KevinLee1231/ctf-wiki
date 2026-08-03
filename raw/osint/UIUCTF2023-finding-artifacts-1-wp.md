# UIUCTF 2023 Finding Artifacts 1 Writeup

## 题目简述

题目要求定位纽约市一尊被称为 “Excellent One” 的青铜雕像所在的博物馆，并提示雕像名称前两个字符为 `ma`、这一形象在南亚十分常见。flag 格式为小写地点名并以下划线分词。

## 解题过程

把材质、别称、城市和名称前缀组合检索，例如：

```text
bronze statue "Excellent One" NYC ma
```

结果指向 `Mahakala Legden (Excellent One)`。Mahakala 是藏传佛教中的护法形象，符合“南亚常见”和 `ma` 前缀两个提示。

继续以完整藏品名称检索博物馆馆藏记录，可以确认这件青铜像属于纽约的 Rubin Museum of Art（鲁宾艺术博物馆）馆藏。关键证据不是名称中碰巧出现 “Excellent One”，而是馆藏条目同时对应：

- 藏品名 `Mahakala Legden (Excellent One)`；
- 材质为青铜；
- 馆藏机构位于纽约市；
- 机构名为 Rubin Museum of Art。

按题目规定格式化机构名称，得到：

```text
uiuctf{rubin_museum_of_art}
```

## 方法总结

这类馆藏定位题应先用引号固定罕见别称，再叠加材质、城市和名称前缀缩小候选。得到对象名称后，要转向博物馆官方馆藏记录核验对象、材质与机构三者的对应关系，不能仅根据搜索摘要或图片标题作结论。
