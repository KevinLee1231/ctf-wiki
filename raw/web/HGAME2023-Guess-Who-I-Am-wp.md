# Guess Who I Am

## 题目简述

站点每轮随机给出一段 Vidar-Team 成员介绍，选手需要提交对应的成员 ID；累计答对 100 次后，计分接口不再返回数字，而是返回 flag。成员资料随前端一起下发，后端只通过客户端 `session` Cookie 保存当前问题和分数，因此可以把“介绍到 ID”的映射整理成本地数据，再依次调用三个 API 自动答题。

出题人公开的[题目源码](https://github.com/ek1ng/My-CTF-Challenges/tree/main/HGAME2023-Guess%20Who%20I%20Am)可用于核对接口和会话实现；下面已经给出完成利用所需的信息。

## 解题过程

### 整理成员映射

前端的 `member.js` 包含每位成员的 `id` 和 `intro`。删除模块导出语句，并移除头像、博客链接等不参与匹配且可能包含 JavaScript 表达式的字段，将剩余数组保存为合法的 `member.json`：

```python
import json

with open("member.json", encoding="utf-8") as file:
    members = json.load(file)
```

真正用于答题的关系只有：

```text
intro -> id
```

### 复现接口流程

开发者工具中可以确认三个接口：

- `GET /api/getQuestion`：返回当前成员介绍；
- `POST /api/verifyAnswer`：表单字段 `id` 为所选成员 ID；
- `GET /api/getScore`：返回当前分数，完成后返回 flag。

问题和分数保存在加密的客户端会话中。每次答对后响应都会通过 `Set-Cookie` 更新 `session`，所以不能始终重放第一次抓到的 Cookie。用 `requests.Session()` 可以自动接收并携带新 Cookie：

```python
import json
import requests

BASE_URL = "http://目标地址"

with open("member.json", encoding="utf-8") as file:
    members = json.load(file)

intro_to_id = {item["intro"]: item["id"] for item in members}
session = requests.Session()

while True:
    question = session.get(
        f"{BASE_URL}/api/getQuestion",
        timeout=10,
    ).json()["message"]

    answer = intro_to_id[question]
    result = session.post(
        f"{BASE_URL}/api/verifyAnswer",
        data={"id": answer},
        timeout=10,
    ).json()["message"]
    print(f"answer: {result}")

    score = session.get(
        f"{BASE_URL}/api/getScore",
        timeout=10,
    ).json()["message"]
    print(f"score: {score}")

    # 正常答题时 message 是整数；完成后它变成 flag 字符串。
    if not isinstance(score, int):
        print(score)
        break
```

原 PDF 中的脚本手动从每个响应提取 `session`，与这里使用 `requests.Session()` 的效果相同。第 100 次答对后，`/api/getScore` 返回：

```text
hgame{Guess_who_i_am^Happy_Crawler}
```

## 方法总结

- 核心技巧：从前端静态资源构造 `intro -> id` 映射，再按真实业务顺序自动调用接口。
- 关键细节：服务端状态封装在会更新的 `session` Cookie 中，脚本必须持续接收 `Set-Cookie`，不能固定使用初始值。
- 复用要点：自动化有状态 Web 流程时，优先使用会话对象，并根据返回值的类型或结构变化判断是否已进入完成分支。
