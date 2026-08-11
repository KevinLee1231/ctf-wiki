# 智商检测鸡

## 题目简述

题目使用 Flask session 记录进度，前端通过四个 JSON API 获取当前状态、领取积分题、提交答案并在完成 100 题后读取 flag。题目内容以 MathML 表示一元一次函数的定积分，需要保持同一会话，解析上下限和系数后自动计算并连续提交。

四个接口为：

- `GET /api/getStatus`：返回当前已完成题数；
- `GET /api/getQuestion`：返回 MathML 题目；
- `POST /api/verify`：接收 JSON 格式的答案；
- `GET /api/getFlag`：完成全部题目后返回 flag。

## 解题过程

浏览器是否能正常渲染 MathML 不影响求解，因为原始 JSON 已包含全部参数。例如题目可能表示 $\int_{-92}^{31}(12x+17)\,dx$。先把负号与紧随其后的数字合并，再依次取出下限、上限、一次项系数和常数项。

下面的脚本保留同一 `requests.Session`，循环获取、计算并验证 100 道题；`BASE_URL` 需要替换为实际实例地址：

```python
import re

import requests
import sympy

BASE_URL = "http://challenge.example"


def calculate(mathml: str):
    normalized = re.sub(
        r"<mo>-</mo>\s*<mn>(\d+)</mn>",
        r"<mn>-\1</mn>",
        mathml,
    )
    lower, upper, coefficient, constant = map(
        int, re.findall(r"<mn>(-?\d+)</mn>", normalized)[:4]
    )
    x = sympy.Symbol("x")
    value = sympy.integrate(
        coefficient * x + constant,
        (x, lower, upper),
    )
    return int(value * 10) / 10


session = requests.Session()
for _ in range(100):
    question = session.get(f"{BASE_URL}/api/getQuestion").json()["question"]
    response = session.post(
        f"{BASE_URL}/api/verify",
        json={"answer": calculate(question)},
    ).json()
    if not response["result"]:
        raise RuntimeError("answer rejected")

status = session.get(f"{BASE_URL}/api/getStatus").json()["solving"]
if status != 100:
    raise RuntimeError(f"unexpected progress: {status}")
print(session.get(f"{BASE_URL}/api/getFlag").json()["flag"])
```

session cookie 是进度状态的绑定点；若每次请求都重新创建客户端，即使计算正确也无法累计到 100。

完成 100 题后，`/api/getFlag` 返回：

```text
hgame{3very0ne_H4tes_Math}
```

官方 PDF 未记录实际返回值，该结果通过 [Zry.IO 的同期 Week 1 复盘](https://zry.io/zh/cybersec/ctf/hgame2021-week-1-writeup/) 补齐；接口状态机与计算方法均已写入正文。

## 方法总结

这类自动答题服务应先从前端脚本还原 API 状态机，再决定是否需要浏览器。JSON 已给出 MathML 时，可直接解析标签并交给符号计算库，不必依赖页面渲染或 OCR。需要连续完成多轮的题目还应优先检查 cookie/session 是否承载进度，并在脚本中复用同一会话。
