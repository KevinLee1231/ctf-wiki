# Nikolai 2

## 题目简述

第二关要求根据 Nikolai 的 OSM 编辑、日记和社交内容确定其具体住宿地址。线索包括 Colorado Springs、最近坐标、步行可达的加油站与 Domino's，以及门外椅子和 lodge。

## 解题过程

检查 `NikolaiM99` 的 OSM changeset 与日记，最新活动落在约：

```text
38.8920, -104.8146
```

再把推文中的环境描述变成地图筛选条件：

- 位于 Colorado Springs；
- 住宿类型是 lodge；
- 步行范围内同时有加油站和 Domino's；
- 街景外部摆有椅子。

符合坐标和全部周边关系的是 `Aspen Lodge`。地图地址为：

```text
3990 N Nevada Ave, Colorado Springs, CO 80907
```

按要求格式化：

```text
UMDCTF-{3990_N_Nevada_Ave_Colorado_Springs_CO_80907}
```

## 方法总结

多条件地理 OSINT 的关键是把叙述转成可验证的空间约束。坐标给出搜索中心，商户组合缩小候选，建筑外观和住宿名称用于最终确认。地址应以多个地图来源的共同字段为准，避免只采信用户评论中的非正式写法。
