# GlacierCTF 2025 Best Food

## 题目简述

题目要求找到 LosFuzzys 喜欢的 Graz 餐厅。附件中的程序故意删去了两个实现，但保留了三类兴趣点、三段距离以及注释：先从 Overpass API 取得 Graz 范围内的全部 `bar`、`atm`、`taxi`，分别求点云中心，再把三个中心作为球面三边定位的参考点。定位出的餐厅还不是最终答案，flag 被放在该地点的大量 Google 评论中。

本题的决定性步骤是从公开地理数据定位实体并关联其公开评论，因此归入 OSINT，而不是按官方仓库的 `misc` 原样放置。

## 解题过程

### 1. 补全 Overpass 查询与中心点计算

对每个类别发起形如下面的 Overpass 查询，范围限定为名称为 Graz 的行政区域：

```text
[out:json];
area["name"="Graz"]->.searchArea;
nwr["amenity"="bar"](area.searchArea);
out center;
```

将 `bar` 依次换为 `atm`、`taxi`。节点直接使用自身经纬度；way 和 relation 使用返回的 `center`。题目所说的 center 是各点经纬度的算术平均值：

$$
\bar\phi=\frac1n\sum_{i=1}^{n}\phi_i,\qquad
\bar\lambda=\frac1n\sum_{i=1}^{n}\lambda_i.
$$

Overpass 数据会随现实设施变化，复现时应保存查询结果；如果中心点已经漂移，不能悄悄改算法，而要使用题目发布时的快照或以仓库 solution 中的自检值为准。

### 2. 在球面上做三边定位

三个中心到目标的 Haversine 距离分别为：

```text
bar  center: 0.6412652119499 km
atm  center: 0.2822972577454 km
taxi center: 0.9572772921063 km
```

地球半径取 $R=6371.0\ \text{km}$。不能把经纬度直接当平面坐标套普通圆交点；应先将每个参考点转换成单位球面上的三维向量

$$
v=(\cos\phi\cos\lambda,\ \cos\phi\sin\lambda,\ \sin\phi),
$$

目标向量 $x$ 与第 $i$ 个参考向量满足

$$
x\cdot v_i=\cos(d_i/R),\qquad \lVert x\rVert=1.
$$

联立三个约束求得目标，再转回经纬度。官方 Rust solver 的 `trilaterate_on_sphere` 同时用 Haversine 和球面余弦距离做了交叉检查，两条路径都收敛到：

```text
47.0666695, 15.4417022
```

### 3. 关联地点评论

将坐标放入地图定位地点，再在该地点的 Google 评论中搜索 `gctf{`、`humorous`、`food review` 等特征词。评论可能被折叠或排序变化，因此坐标才是稳定的定位结果；仓库中的出题 flag 文件也对评论内容提供了离线佐证：

```text
gctf{g00d f00d, hum0r0u5 f00d r3v13w5}
```

## 方法总结

这道题的证据链是“源码中的类别和距离 → Overpass 公开兴趣点 → 球面三边定位 → 地图地点 → 公开评论”。关键坑有两个：一是 way/relation 的坐标要取 `center`，二是距离不到 1 km 也不代表可以忽略题目明确要求的球面模型。由于在线 POI 和评论都可能变化，严谨的复现应记录查询时间与原始响应，并用仓库中固定的目标坐标和 flag 做最终校验。
