# 留言板（粉）

## 题目简述

页面接收原始 XML 请求体并使用 PHP `DOMDocument` 解析。关键调用为 `loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD)`：`LIBXML_DTDLOAD` 允许加载 DTD，`LIBXML_NOENT` 会展开实体。解析后的 `textContent` 又被回显，因此攻击者可以定义指向本地文件的外部实体并读出 `/flag`。

登录页存在固定挑战凭据 `admin / admin123`，但 XML 页面没有会话校验；它只影响正常页面导航，不是 XXE 成立的条件。

## 解题过程

### 漏洞机制

XML 的 DTD 可以声明实体。内部实体只替换固定文本，而带 `SYSTEM` 标识的外部实体可以引用 URI，例如本地文件：

```xml
<!ENTITY xxe SYSTEM "file:///flag">
```

安全配置通常应禁止外部 DTD 与实体解析；本题显式打开了加载和替换选项，并把展开后的 DOM 文本返回给客户端，形成可直接回显的 XXE 文件读取。

### 构造请求

向 XML 接口发送以下原始请求体，`Content-Type` 使用 `application/xml`：

```xml
<?xml version="1.0"?>
<!DOCTYPE message [
  <!ENTITY xxe SYSTEM "file:///flag">
]>
<message>&xxe;</message>
```

解析器读取 `/flag`，将内容替换到 `&xxe;`，随后 `dom->textContent` 把结果回显。Dockerfile 也确认 flag 文件就位于该绝对路径。

## 方法总结

- 核心技巧：利用启用了 DTD 加载与实体替换的 XML 解析器读取本地文件。
- 识别信号：用户可控原始 XML、`LIBXML_DTDLOAD`、`LIBXML_NOENT` 与解析结果回显同时出现。
- 复用要点：判断 XXE 时要同时确认外部实体能否加载、实体是否展开、结果是否可见；仅看到 XML 输入并不足以断定漏洞可利用。
