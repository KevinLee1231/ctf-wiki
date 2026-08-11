# legendary

## 题目简述

应用展示动物信息，并以 `X-Signature` 保护 `/api/animal/<name>` 及带 fields 的变体。签名是 `hash_hmac('sha256', $api, APP_SECRET_KEY)`，密钥在 `config.php` 中构建时替换为随机 32 位十六进制值。题目同时存在两条由 NGINX、PHP URL 规范化及 MySQL 标识符转义相互叠加形成的漏洞链：先读取 `config.php` 得到 HMAC 密钥，再对已签名的 fields 接口实施 SQL 注入，使 PHP 的 `glob://` 流包装器枚举出随机化的 flag 文件名，最后再次利用静态文件路径穿越读取内容。

## 解题过程

### 泄露签名密钥

静态文件 location 只打算提供压缩资源，但 `try_files` 直接把正则捕获的路径拼入文件名：

```nginx
location ~ /static/(.*)\.(\w+) {
    try_files /static/$1.min.$2 /static/$1.$2 =404;
}
```

因此以保留 `..` 的原始路径请求 `/static/../config.php` 时，第二个候选会在文件系统规范化后命中 Web 根目录下的 `config.php`。从响应取得 `APP_SECRET_KEY`，不要使用客户端自动折叠路径的普通请求方式。

### 规范化后签名并注入 fields

PHP 在进入路由前还会对 `PATH_INFO` 做一次 `urldecode`：

```php
$api = urldecode(parse_url($_SERVER['PATH_INFO'])['path']);
```

所以签名对象不是浏览器中原始的百分号编码 URL，而是 PHP 最终传给 `do_api()` 的 API 字符串。官方解题脚本对构造的 payload 做两次 URL 解码再计算 HMAC；复现时必须以实际部署的 NGINX/PHP 解码层数为准，否则会得到 403。

fields 路由把逗号分隔字段逐一包上反引号：

```php
$fieldstr = join(',', array_map('db_escape_col', explode(',', $fields)));
$animal = db_query_one("SELECT $fieldstr FROM animals WHERE name = ?", [$name]);
```

`db_escape_col` 只把反引号加倍，遗漏了 MySQL 对反斜杠、NUL 和 `#` 注释的解析语义。官方 payload 在 fields 段放入双重编码的 `\?`、`%00` 和 `%23`，令反斜杠逃逸程序补上的反引号、NUL 截断余下输入、`#` 注释掉末尾反引号；由此逃出标识符上下文。注入的派生表为 `imagedir` 赋值 `0x676c6f623a2f2f2f662a`，即 `glob:///f*`。

```text
/api/animal/<经编码的注入 name>/fields/<经编码的 fields 注入>
X-Signature: HMAC-SHA256(规范化后的 API 路径, APP_SECRET_KEY)
```

服务端在结果含 `imagedir` 时会调用 `scandir()` 并返回第一个目录项：

```php
$animal['imagedir'] = $animal['imagedir'] . '/'
    . array_values(array_diff(scandir($animal['imagedir']), ['.', '..']))[0];
```

`glob:///f*` 因而枚举出容器根目录中形如 `flag-<随机十六进制>.txt` 的文件名。响应中只需提取这一文件名，不必猜测随机后缀。

### 读取 flag

已知文件名后，再以原始 HTTP 请求访问静态 location 下经多个 `../` 回到根目录的路径。该 location 的同一 `try_files` 拼接会解析目录段并命中 `/flag-<随机十六进制>.txt`。使用能保留原始 request target 的客户端，避免 URL 库在发送前归一化 `..`；读取响应正文即可。

### 验证

完整链必须依次验证三个中间结果：读取到随机 HMAC key、签名 fields 请求返回随机 flag 文件名、静态路径请求返回 flag 内容。官方题目给出的结果是：

```text
DUCTF{o.o_what_a_great_bit_of_ngineering}
```

## 方法总结

- 核心技巧：先用 NGINX 静态路径解析泄露 HMAC key，再利用 MySQL 标识符转义缺陷与 PHP `glob://` 枚举不可预测文件名。
- 识别信号：受签名保护的 API 若含有可读密钥文件的路径穿越，签名并非安全边界；动态字段列表即使尝试 quote，也要按目标数据库的完整词法规则检查。
- 复用要点：多层 URL 解码、客户端规范化和服务端路径解析必须分别建模。对字段名应使用固定 allowlist，静态资源映射不能让用户输入参与文件系统路径拼接。
