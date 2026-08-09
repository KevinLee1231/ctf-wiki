# Tail

## 题目简述

附件只展示飞机尾翼，要求识别航空公司及其主要枢纽机场的三字 IATA 代码。尾翼上的白色花朵标志是决定性视觉线索。

## 解题过程

观察尾翼的蓝色底色和白色大溪地栀子花图案：

![蓝色尾翼上的白色大溪地栀子花标志，用于识别 Air Tahiti Nui](n00bzCTF2024-tail-wp/air-tahiti-nui-tail-logo.jpg)

按 `blue airline tail white flower logo` 等特征搜索航空公司标志，可以识别为 Air Tahiti Nui。该公司的主要枢纽是法属波利尼西亚的 Faa'a International Airport，其 IATA 代码为 `PPT`。因此：

```text
n00bz{PPT}
```

## 方法总结

飞机涂装识别应先提取颜色、图形和地域意象，再用航空公司资料核对。题目要的是枢纽机场代码，不是航空公司代码或城市名称。
