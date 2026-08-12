# DownUnderCTF 2022 Honk Honk Writeup

## 题目简述

题目给出澳大利亚车牌 `23HONK`，要求查询该车的 CTP 到期日期，格式为 `DD/MM/YYYY`。这是车辆公开记录检索题，关键是选择车牌所属司法辖区的官方查询入口。

题目发布时可通过 New South Wales 的免费车辆注册查询服务获取注册状态、车辆描述和到期信息。由于车辆注册会续期、转移或失效，当前查询结果不一定还能复现 2022 年比赛时的页面状态。

## 解题过程

进入 [NSW Free Rego Check](https://free-rego-check.service.nsw.gov.au/)，以 NSW 车牌 `23HONK` 查询。比赛时返回的关键字段包括：

```text
Plate: 23HONK
Vehicle: NISSAN MARCH 3 DOOR TURBO SEDAN
CTP/registration due date: 19/07/2023
```

本题只要求日期，因此按指定格式提交：

```text
DUCTF{19/07/2023}
```

车辆描述应一并记录，因为它会成为后续 `Does It Fit My CTF?` 的主要搜索线索。

## 方法总结

车辆 OSINT 首先要确认国家、州和车牌格式，再优先使用对应政府的官方注册查询。动态公共记录具有时间性，WP 应保存当时决定答案的字段，而不能只留一个查询链接；否则车辆状态更新后，历史解法将无法复现。
