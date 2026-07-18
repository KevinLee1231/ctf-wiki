# week2edr

## 题目简述

题目复现了一类 PHP `extract()` 变量覆盖漏洞。入口把用户可控的 `$_REQUEST` 传给 `$show_form`；函数内部直接执行 `extract($params)`，默认使用 `EXTR_OVERWRITE`，可覆盖当前作用域中已有的变量。被覆盖的 `$strip_slashes` 随后又被当作变量函数调用，因此攻击者可以把它改成 `system`，将另一个请求参数变成系统命令。

## 解题过程

原本的 `$strip_slashes` 是一个闭包：

```php
$strip_slashes = function($var) {
    if (!get_magic_quotes_gpc()) {
        return $var;
    }
    return stripslashes($var);
};
```

`$show_form` 通过引用捕获它，但进入函数后又把请求参数展开到局部变量：

```php
$show_form = function($params) use (&$strip_slashes, &$show_input) {
    extract($params);
    $host  = isset($host)  ? $strip_slashes($host)  : "127.0.0.1";
    $path  = isset($path)  ? $strip_slashes($path)  : "";
    $row   = isset($row)   ? $strip_slashes($row)   : "";
    $limit = isset($limit) ? $strip_slashes($limit) : 1000;
    /* 省略表单输出 */
};

$show_form($_REQUEST);
```

PHP 允许用字符串变量调用同名函数：若 `$strip_slashes = 'system'`，表达式 `$strip_slashes($host)` 就等价于 `system($host)`。因此请求可写成：

```text
/?strip_slashes=system&host=cat%20/flag
```

`extract($_REQUEST)` 先把 `$strip_slashes` 覆盖为字符串 `system`，同时把 `$host` 设为 `cat /flag`；下一行调用变量函数时执行该命令，标准输出进入 HTTP 响应。

## 方法总结

- 核心技巧：利用 `extract()` 默认覆盖已有变量，把闭包变量改成危险的变量函数名。
- 识别信号：用户数组直接进入 `extract`，随后存在 `$variable($argument)` 形式的函数调用。
- 复用要点：必须沿数据流同时确认“函数名变量”和“函数参数”都可控；防御时避免对请求数据使用 `extract`，或至少使用 `EXTR_SKIP` 和明确白名单。
