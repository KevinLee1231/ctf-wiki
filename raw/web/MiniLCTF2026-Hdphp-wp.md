# Hdphp

## 题目简述

应用把 GET 参数 `f` 直接传给 PHP `include`，只以如下正则过滤协议名和若干目录名：

```php
if (isset($_GET['f']) && !preg_match('/flag|file|php|data|zip|phar|(proc|dev|bin|usr|var).{15,}/i', $_GET['f'])) {
    usleep(200000);
    include $_GET['f'];
}
```

容器启动时将 flag 写到 `/flag` 并清除环境变量。漏洞不是 PHP wrapper，而是 Nginx 处理大请求体时生成的临时文件可经 `/proc/<pid>/fd/<fd>` 引用；`include` 会把其中的 PHP 片段作为代码解释。过滤条件只限制 `proc` 之后的字符数，且 200 ms 延迟要求可靠保持临时文件存活。

## 解题过程

### 定位可包含的临时文件

向 Nginx 持续上传超过 `client_body_buffer_size` 的大请求体时，Nginx 会将请求体落入 client-body 临时文件。目标 PHP-FPM/Nginx 进程打开该文件后，Linux 的 `/proc/<pid>/fd/<fd>` 软链接仍可引用这个打开的 inode，即便目录项随后被删除。

直接使用 `/proc/<pid>/fd/<fd>` 会受到两层阻碍：正则会拒绝 `proc` 后面过长的内容，而 PHP 路径处理会因为 fd 链接的 `(deleted)` 目标而失败。原题解给出的绕过是 `/proc/<pid>/fd/a/../<fd>`：冗余的 `a/..` 让原始字符串仍满足“`proc` 后至多 14 个字符”的限制，同时改变 PHP 在真正调用 `open` 前的路径规范化分支。这里不能理解成 Linux 内核能直接穿过一个不存在的 `a` 目录；关键是 PHP 先做词法归一化，再把折叠后的 fd 路径交给内核。该行为依赖具体 PHP 构建，实际 PID 与 fd 也需针对靶机枚举。

### 竞态包含

请求体开头放入短 PHP 代码，后面填充大量无害数据以延长上传与临时文件生存时间；同时并发请求包含候选 fd。下列是流程骨架，变量均需根据本地靶场的 PID、fd 范围及 Nginx 配置填写：

```text
线程 A：持续 POST 大正文
  正文开头：<?php echo file_get_contents('/flag'); ?>
  正文后部：足够大的填充字节

线程 B：在每个上传尚未结束时并发请求
  GET /?f=/proc/<php-fpm-pid>/fd/a/../<candidate-fd>
```

`usleep(200000)` 位于 `include` 前，短请求会使临时文件在真正打开前消失，因此不能把它当作普通单次 LFI。保持大正文在传输中，并用多个 worker/候选 fd 重试，才能让文件在延迟后仍被包含。包含成功时，正文开头的 PHP 被解释，响应输出 `/flag` 内容。

### 验证

验证不是只看到 `include` 报错，而是确认响应出现由请求体中 `echo` 产生的受控标记；把标记替换为读取 `/flag` 后得到最终输出。这里没有运行或攻击比赛实例，步骤依据给出的 `index.php` 与入口脚本核对。

## 方法总结

- 核心技巧：将 Nginx 大请求体临时文件经 `/proc/<pid>/fd` 引入 PHP `include`，把 LFI 升级为代码执行。
- 识别信号：可控 `include`、关键词式路径黑名单、Nginx/PHP-FPM 组合，以及 `usleep` 等专门扩大竞态难度的延迟。
- 复用要点：计算正则是按原字符串还是规范化路径匹配；同时验证 PHP 的路径处理与内核 `open` 解析。修复应移除用户可控 `include`，或把标识映射到固定的允许列表文件。
