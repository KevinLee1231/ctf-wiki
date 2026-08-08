# miniLCTF 2024 乐到不跑 Writeup

## 题目简述

网页通过浏览器地理位置持续上报跑步轨迹。后端随机选取 3 个检查点，要求总里程超过 10 km、全部检查点均被访问、相邻上报间隔在 1 到 10 秒之间、单步不超过 100 m、瞬时速度不超过 40 m/s，且最终平均速度大于 10 m/s。完成后 `/status` 才返回环境变量中的 flag。

## 解题过程

### 从接口文档还原判定条件

前端 `main.js` 注释提示 `/docs`，FastAPI 自动文档列出：

```text
GET  /restart
GET  /status
GET  /checkpoints
POST /location    {"lat": ..., "lon": ...}
```

源码中的关键限制为：

```text
1 s <= time_delta <= 10 s
distance_per_update <= 100 m
velocity <= 40 m/s
all 3 checkpoints visited within 50 m
distance_sum > 10000 m
average_velocity > 10 m/s
```

直接伪造一次 10 km 跳跃会触发作弊。应在检查点之间插值，每 3 秒移动约 95～99 m；这样瞬时速度约 32 m/s，同时循环多圈累积里程。

### 生成合法轨迹

```python
import math
import time
import requests

BASE = "http://challenge.example"

def distance(a, b):
    r = 6_371_000.0
    p1, p2 = math.radians(a[0]), math.radians(b[0])
    dp = math.radians(b[0] - a[0])
    dl = math.radians(b[1] - a[1])
    h = math.sin(dp/2)**2 + math.cos(p1)*math.cos(p2)*math.sin(dl/2)**2
    return 2*r*math.atan2(math.sqrt(h), math.sqrt(1-h))

def interpolate(a, b, step=95.0):
    total = distance(a, b)
    pieces = max(1, math.ceil(total / step))
    for i in range(pieces):
        t = i / pieces
        yield (a[0] + t*(b[0]-a[0]), a[1] + t*(b[1]-a[1]))

requests.get(BASE + "/restart").raise_for_status()
raw = requests.get(BASE + "/checkpoints").json()["checkpoints"]
points = [(x["lat"], x["lon"]) for x in raw]

while True:
    for i, start in enumerate(points):
        end = points[(i + 1) % len(points)]
        for lat, lon in interpolate(start, end):
            requests.post(BASE + "/location", json={"lat": lat, "lon": lon}).raise_for_status()
            state = requests.get(BASE + "/status").json()
            if state["status"] == "finished":
                print(state["flag"])
                raise SystemExit
            if state["status"] in {"timeout", "cheating"}:
                raise RuntimeError(state)
            time.sleep(3)
```

题目官方脚本使用 99 m 步长；这里留出 5 m 浮点和球面距离余量。由于后端状态是全局变量而非按用户隔离，复现期间不要并发调用 `/restart`，否则会清空当前轨迹。

## 方法总结

这不是篡改单个坐标即可完成的签到题，而是对服务端状态机的精确模拟。先从源码列出时间、距离、速度、检查点和累计里程五类约束，再选择同时满足全部不等式的上报周期与步长。面对位置校验，应优先构造连续合法轨迹，避免只绕前端定位 API。
