# Petrol Head

## 题目简述

题面只给出车牌 `MJ05 XRL`，要求查询车辆 registration 到期日并按 `MM/DD/YYYY` 提交。牌照格式指向英国，需要使用英国官方车辆查询服务。

## 解题过程

英国现行式车牌通常由两字母、两数字和三字母组成，`MJ05 XRL` 符合这一格式。进入英国政府的 [DVLA 车辆信息查询入口](https://www.gov.uk/get-vehicle-information-from-dvla)，输入 registration number，并按页面要求确认车辆信息。

DVLA 页面能够显示 vehicle tax 的状态与到期日。BYUCTF 2024 比赛期间，该车牌的查询快照显示到期日为：

```text
1 January 2025
```

转换为题目格式：

```text
byuctf{01/01/2025}
```

这是动态车辆记录；2025 年后重新查询可能显示续期、SORN 或其他新状态，不能用当前值覆盖比赛时的历史答案。

## 方法总结

先用牌照结构确定司法辖区，再优先查政府一手数据库。OSINT WP 必须记录查询字段、时间语境和格式转换；对会续期的注册记录，要明确答案对应比赛快照，而非永久不变的属性。
