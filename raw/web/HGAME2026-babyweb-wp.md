# babyweb?

## 题目简述

题目给出的 PHP 上传逻辑允许上传 `.php` 文件。上传处理脚本会把原文件名直接拼到 `uploads/` 目录下保存，允许扩展名列表中也包含 `php` 和 `htaccess`，因此可以直接上传 WebShell。拿到当前 Web 容器权限后，flag 不在当前容器内，需要继续做内网探测，发现同网段还有 Next.js 服务。

关键上传逻辑如下：

```php
$target_dir = "uploads/";
$origName = $_FILES['fileToUpload']['name'];
$target_file = $target_dir . $origName;
move_uploaded_file($_FILES['fileToUpload']['tmp_name'], $target_file);

$fileExt = strtolower(pathinfo($_FILES["fileToUpload"]["name"], PATHINFO_EXTENSION));
$allowedTypes = ["jpg", "jpeg", "png", "gif", "pdf", "doc", "docx", "txt", "htaccess", "php"];
```

## 解题过程

先上传一个简单 PHP WebShell。由于服务端允许 `.php`，文件名也没有做随机化或危险扩展过滤，访问 `uploads/<filename>.php` 即可执行命令。

进入容器后没有直接找到 flag。继续查看网络环境，当前容器处在 `10.0.0.1` 附近的内网段，扫描后发现 `10.0.0.2:3000` 运行着 Next.js 服务。

后续利用 Next.js 的 CVE-2025-55182 实现 RCE，在内网服务上执行命令读取 flag。这里 PHP WebShell 的作用是进入第一层容器并完成内网探测，真正拿 flag 的点在第二层 Next.js 服务。

## 方法总结

- 核心技巧：危险文件上传获得第一层命令执行，再进行容器内网探测并利用内网 Next.js RCE。
- 识别信号：上传白名单里直接出现 `php`，且保存文件名可控时，优先尝试 WebShell；当前容器没有 flag 时继续看内网。
- 复用要点：多容器 Web 题要把“上传点”和“最终 flag 所在服务”分开看，第一层 Shell 常常只是跳板。
