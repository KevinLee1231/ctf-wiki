# Under the Overpass

## 题目简述

题面要求在柏林找出一座一侧紧邻牙医、另一侧紧邻酒店的外国使馆，并统计其 100 米内的监控设备数量。给定工具是 [Overpass Turbo](https://overpass-turbo.eu/)；答案格式为国家名加摄像头数量。

## 解题过程

先查询柏林所有使馆和外交办公室：

```overpassql
[out:json][timeout:25];
{{geocodeArea:Berlin}}->.searchArea;
(
  nwr["amenity"="embassy"](area.searchArea);
  nwr["office"="diplomatic"](area.searchArea);
);
out center;
```

逐个检查候选周边的 `amenity=dentist` 与酒店 POI。唯一满足牙医和酒店分居两侧关系的目标是 El Salvador Embassy，中心坐标约为 `52.5287089, 13.3800402`。

以该点为中心查询 100 米内所有可能的监控标注：

```overpassql
[out:json][timeout:25];
(
  nwr(around:100,52.5287089,13.3800402)["man_made"="surveillance"];
  nwr(around:100,52.5287089,13.3800402)["surveillance"];
);
out body;
```

比赛时的 OpenStreetMap 快照返回 10 个对象，因此：

```text
byuctf{El_Salvador:10}
```

OSM 是动态数据源，当前查询数量可能变化；复核历史题目时应记录查询、坐标和比赛时结果，而不能只保存最终数字。

## 方法总结

- 核心技巧：用 Overpass 先枚举外交机构，再以相邻 POI 关系筛选，最后做半径空间查询计数。
- 识别信号：题面给出多种“紧邻/另一侧/范围内”条件时，适合用 OSM 标签和空间关系逐层收敛。
- 复用要点：查询应覆盖 node、way、relation，避免只查节点；动态地图结论要保存坐标、查询语句和时间快照。
