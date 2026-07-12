# Make PHP Great Again

## 题目简述

题目利用新版 PHP 中 `require_once` 对已包含文件的 once 记录和真实路径/hash 匹配细节。若被包含目标经过很多层软链接或 `/proc/self/root` 这类路径绕行，`require_once` 的 once 判断可能失效，从而让同一实际文件被重复包含。解法通过超长路径链配合 `php://filter/read=convert.base64-encode/resource=...` 读取目标 PHP 文件内容。

## 解题过程

PHP 最新版的一个小 trick：`require_once` 包含的软链接层数较多时，once 的 hash 匹配会失效，造成重复包含。原始 WP 没有展开源码分析，但利用点可概括为：

1. `require_once` 不是只按用户输入字符串判断是否已包含，而会涉及规范化后的路径/文件标识。
2. 多层软链接或 `/proc/self/root` 路径绕行会让同一实际文件呈现为不同路径。
3. 一旦 once 去重失败，就可以重复触发包含/读取链条。

Payload 形态如下，实际使用时把 `<target-host>` 替换为题目地址，并按目标路径调整重复层数：

```plain
http://<target-host>/?file=php://filter/read=convert.base64-encode/resource=file:///proc/self/root/proc/self/root/.../var/www/html/flag.php
```

## 方法总结

- 核心技巧：通过软链接/`/proc/self/root` 多层路径绕过 `require_once` 的 once 去重，再配合 `php://filter` 读取 PHP 源码或 flag 文件。
- 识别信号：题目使用 `require_once`、路径由用户控制、存在软链接或 Linux `/proc` 路径可用时，应检查真实路径规范化和 once cache 是否可绕过。
- 复用要点：不要在 WP 中保留一次性靶机 URL；保留 wrapper、路径绕行结构和目标文件位置即可复现思路。
