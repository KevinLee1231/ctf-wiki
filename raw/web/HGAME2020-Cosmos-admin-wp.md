# Cosmos 的博客后台

## 题目简述

博客后台把文件包含、调试代码、弱类型比较和服务端取图串成了一条利用链。目标不是在某一个页面直接读取 flag，而是先通过源码泄露取得管理员凭据，再绕过登录，最后滥用 cURL 的 `file://` 协议读取服务器本地文件。

## 解题过程

首页的 `action` 参数会被 PHP 当作文件包含目标。使用 `php://filter` 对目标源码做 Base64 编码，可以避免源码被解释执行：

```text
?action=php://filter/convert.base64-encode/resource=login.php
```

解码后可见调试逻辑最终执行：

```php
$v = $_GET['debug'];
eval("var_dump($$v);");
```

令 `debug=GLOBALS`，`${GLOBALS}` 就会把全局变量整体打印出来，其中包含 `$admin_username` 和 `$admin_password`。登录判断还对用户输入的 MD5 值使用了 PHP 弱类型比较，因此可提交一个摘要形如 `0e<纯数字>` 的魔术哈希，例如：

```text
aabg7XSs
```

该字符串的 MD5 会被解释为科学计数法中的零，从而与同类摘要在 `==` 比较中相等。进入后台后，`insert_img` 功能使用 cURL 获取用户指定 URL，并把响应 Base64 编码后嵌入图片。虽然代码限制了主机名，但 `file://localhost/` 仍满足本地主机条件：

```text
file://localhost/flag
```

取出返回内容中的 Base64 数据并解码，即可得到 flag。

## 方法总结

- 核心链路：`php://filter` 读取源码、`GLOBALS` 泄露凭据、MD5 魔术哈希绕过弱比较、cURL `file://` 读取本地文件。
- 识别信号：动态包含参数、可控调试变量、PHP `==` 比较和服务端代取 URL 同时出现时，应检查能否组合利用。
- 复用要点：协议白名单比单纯的主机名校验更重要；服务端请求功能若允许 `file://`，即使不能访问外网也可能造成任意文件读取。

> 外部复现资料补充了原 PDF 未完整展示的参数细节；本文已把利用条件与链路完整写入正文。参考：[HGame 2020 Web 复现记录](https://kaze0912.github.io/2020/01/31/hgame2020-web-wp/)。
