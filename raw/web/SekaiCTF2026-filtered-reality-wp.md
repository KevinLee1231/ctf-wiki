# Filtered Reality

## 题目简述

目标是 WordPress 6.6.2/PHP 8.3 的 records bureau。flag 由独立 UID 的 keeper 守护进程持有，Web 用户无法直接读取；管理员 bot 会审查被 seal 的记录。官方题解要求完成五段缺一不可的 client→server 利用链。

公开订阅者账号为：

```text
archive_clerk / ArchiveClerk!2026
```

## 解题过程

### 1. 伪造 screen id，取得 seal nonce

`archive_seal_record` 没有 capability 检查，只验证 nonce。nonce 只在当前 screen 为 `toplevel_page_archive-desk` 时输出，而 WordPress 从 `PHP_SELF` 推导 screen。

订阅者请求：

```http
GET /wp-admin/index.php/x%0A/wp-admin/toplevel_page_archive-desk
```

换行进入 `PATH_INFO` 后，WordPress 错误地把订阅者可访问的 dashboard 识别为 Reviewer Desk，于是 `admin_enqueue_scripts` 钩子在响应中打印 `archive_seal_record` nonce。再以该 nonce 调用 `admin-ajax.php?action=archive_seal_record`，即可把指定记录引用写入 bot 的审核队列；这里仍没有获得管理员权限，只得到“让 bot 打开记录”的能力。

### 2. SXG fallback、DOM clobber 与 CSP nonce 泄露

`/?render=<ref>` 原样回显记录正文，还把导航请求的 `Accept` 反射进 `Content-Type`。Chrome 会把它作为 Signed HTTP Exchange 解析，普通脚本无法执行。

构造以 `sxg1-b3\x00` 开头的无效 SXG，后接两字节大端 fallback URL 长度，并把 HTTPS fallback 指向内部
`https://127.0.0.1:1338/?archive_moderation=1&ref=<ref>`。
SXG 校验失败后，Chrome 导航到该 fallback；重新导航时的 `Accept` 不再包含 signed-exchange，moderation 页面才会按 HTML 正常渲染。

moderation 页面用自制递归 sanitizer 处理记录的 `source`。把 payload 放进
`<form><input name="childNodes"><input name="childNodes">...</form>`
后，表单的 named property 会遮蔽真正的 `form.childNodes`，递归器只遍历被命名的 input，从而漏掉同级的 `<style>` 或 `<iframe>`。

CSP nonce 由 `substr(SHA256(wp_salt || floor(time/1800)), 0, 24)` 生成，是 24 个十六进制字符。浏览器会隐藏脚本元素的 nonce 属性，但页面又把同一值放进
`<meta id="csp-nonce" content="...">`，CSS 属性选择器仍可读取它。注入的 `<style>` 用同源 `@import` 加载包含前缀、后缀和三字符子串选择器的 CSS，并把命中结果请求到 `/?archive_leak`；根据首尾字符和全部 trigram 即可重建 nonce。随后再利用同一 clobber 注入：

```html
<iframe srcdoc="<script nonce=NONCE src=/?archive_raw=exp></script>">
```

脚本在管理员上下文中运行，读取 `archive_process_record` nonce 并提交序列化对象。

### 3. PHP 对象注入

`archive_process_record` 对输入执行：

```php
@unserialize(base64_decode($blob))
```

`SecureTableGenerator::__wakeup` 会清理 `data` 和 `headers`，却遗漏 `allowedTags`。后续对对象执行宽松 `in_array` 比较，触发 `WP_HTML_Tag_Processor::__toString()`，并沿 WordPress core gadget：

```text
WP_HTML_Tag_Processor
  -> WP_Block_List::offsetGet
  -> WP_Block::__construct
  -> WP_Block_Patterns_Registry::get_content
  -> include($filePath)
```

`WP_HTML_Token::__destruct` 看似也能形成链，但在 WordPress 6.6.2 中其 `__wakeup` 会抛异常；PHP 8.3 不会再对这种未成功反序列化的对象调用析构，因此它只是诱饵。

### 4. `php://filter` RCE

把 `$filePath` 设置为 `php://filter` 的 iconv/base64 链，从空的 `php://temp` 合成 PHP 载荷并被 `include`，取得 `www-data` 代码执行。由于输出会被 WordPress 缓冲，载荷把结果写入可访问的 `wp-content/uploads/<token>.log`。

### 5. keeper 长度扩展

keeper 的签名为：

$$
\operatorname{SHA256}(\text{SECRET}\parallel m).
$$

`SECRET` 是 keeper 启动时生成、只保存在内存中的 16 个随机字节。`SIGN` 拒绝包含 `give_flag` 的消息，而 `FLAG` 还要求消息包含当前 60 秒 cycle。先通过本地 Unix socket 查询 cycle，再签名合法消息：

```text
cycle=<n>;user=guest
```

再利用 SHA-256 长度扩展，在不知道 SECRET 的情况下追加：

```text
;cmd=give_flag
```

长度扩展需要原消息前未知前缀长度；即使不知道源码中的 16，也可像官方脚本一样枚举 $0$ 至 $48$。把
`m || SHA256-padding || ";cmd=give_flag"`
和继续压缩得到的新摘要交给 `FLAG`，命中正确长度时 keeper 返回 flag，RCE 载荷将响应写入 uploads。

最终得到：

```text
SEKAI{th3_d4y_n3v3r_3nds_1f_y0u_r34d_f4st}
```

## 方法总结

完整链条是 screen-id spoof → SXG fallback → DOM clobber/CSP nonce leak → PHP POP → `php://filter` RCE → secret-prefix hash length extension。每一段只提供下一段所需能力：订阅者先取得队列写入权，bot 上下文再取得管理员 action，Web RCE 最后才能访问本地 keeper。审计长链题时应把每一步的输入能力与输出能力写清，否则很容易误把“能让 bot 打开页面”当成已获得 XSS，或把 `www-data` RCE 当成已经能读跨 UID 的 flag。
