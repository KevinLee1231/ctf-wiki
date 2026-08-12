# DownUnderCTF 2020 - I think this one is really going to take off

## 题目简述

题面给出三条公开情报线索：2020 年 9 月 1 日、一架“大型美国空中加油机”、以及澳大利亚地标 “boxing croc”。目标是先从历史航迹定位飞机注册号，再查该机首次飞行日期，按 `DUCTF{DD-MM-YY}` 提交。

## 解题过程

### 固定地点与候选机型

搜索 “boxing croc” 可定位 Boxing Crocodile：地址为 326 Arnhem Highway, Humpty Doo, Northern Territory。题面中的 American refuelling plane 则把候选范围收敛到美国军用 tanker；历史 ADS-B 列表中常见的 `K35R` 代码对应 KC-135R。

### 检查指定日期的历史航迹

在支持历史记录的 ADS-B 数据源中筛选 2020-09-01 澳大利亚区域。当天列出的美国军机只有 5 架，其中 4 架是 `K35R`。逐一查看轨迹，只有注册号 `58-0086` 的航迹从 Humpty Doo 经过，并最接近 Boxing Crocodile。

官方 writeup 记录的历史轨迹页参数为：

```text
AircraftID=11404929
datum=20200901
```

这些参数比泛泛的搜索结果更能固定“哪一天、哪一架”的证据链。

### 从注册号查首次飞行

再以精确注册号 `58-0086` 查询飞机资料库。[PlaneFinder 的 58-0086 资料页](https://planefinder.net/data/aircraft/58-0086)记录该 KC-135 的 first flight 为 16 July 1959。按题目规定格式化：

```text
DUCTF{16-07-59}
```

## 方法总结

- 核心技巧：地点消歧 → 指定日期历史 ADS-B 筛选 → 航迹空间比对 → 注册号反查机身履历。
- 识别信号：题面同时给出交通工具类型、地标和日期时，应优先寻找有历史数据的运输追踪站，而不是只用实时地图。
- 复用要点：先用 registration/ICAO ID 固定实体，再查首次飞行等静态属性；军机可能在部分平台被过滤，应交叉核对多个公开航迹或机身数据库，并记录日期与轨迹参数。
