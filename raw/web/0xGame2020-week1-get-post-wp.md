# week1get&post

## 题目简述

题目要求在同一个 HTTP 请求中同时提供 GET 查询参数和 POST 表单参数。服务端使用 PHP 的严格比较 `===`，只有两个参数都存在且都与固定字符串完全相同才会输出 flag。

```php
include("flag.php");
highlight_file(__FILE__);

if (isset($_GET['0xGame']) && isset($_POST['X1cT34m'])) {
    $a = $_GET['0xGame'];
    $b = $_POST['X1cT34m'];
    $c = 'acd666tql';
    if ($a === $c && $b === $c) {
        echo $flag;
    }
} else {
    die("Do you konw GET & POST ?");
}
```

## 解题过程

GET 参数位于 URL 查询串中：

```text
/?0xGame=acd666tql
```

POST 参数放在请求体中，并使用默认的表单编码：

```text
X1cT34m=acd666tql
```

用 `curl` 可以一次构造完整请求：

```bash
curl -s -X POST \
  -d 'X1cT34m=acd666tql' \
  'http://<HOST>:<PORT>/?0xGame=acd666tql'
```

`-d` 会生成 `application/x-www-form-urlencoded` 请求体。服务端分别从 `$_GET` 和 `$_POST` 取值，两个严格比较均成立后返回 flag。

## 方法总结

- 核心技巧：在一个请求中组合 URL 查询参数与 POST 表单数据。
- 识别信号：源码同时访问 `$_GET` 和 `$_POST`，且用 `isset` 要求两者都存在。
- 复用要点：严格比较要求参数名、大小写和字符串内容完全一致；GET 与 POST 同名也不会自动互相替代。
