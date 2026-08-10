# Sign In

## 题目简述

题目直接给出一段 Base64 文本：

```text
aGdhbWV7V2VsY29tZV9Ub19IR0FNRTIwMjMhfQ==
```

决定性步骤是识别并解码表示层编码，因此归入 Crypto，而不是沿用原 PDF 的宽泛 Misc 分类。

## 解题过程

Base64 常见特征包括字符集只含大小写字母、数字、`+`、`/`，并可能以 `=` 补齐。使用标准库解码：

```python
import base64

encoded = "aGdhbWV7V2VsY29tZV9Ub19IR0FNRTIwMjMhfQ=="
print(base64.b64decode(encoded).decode())
```

得到：

```text
hgame{Welcome_To_HGAME2023!}
```

## 方法总结

- 核心技巧：根据字符集和末尾填充识别 Base64，并还原原始字节。
- 复用要点：Base64 是编码而不是加密；解码后还应检查输出是否为完整文本、文件头或下一层编码。
