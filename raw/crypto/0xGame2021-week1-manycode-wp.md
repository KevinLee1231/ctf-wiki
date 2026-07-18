# week1manycode

## 题目简述

附件是一段 AAEncode 风格的 JavaScript，解开后还有 Base16、Base32、Base64 三层编码。目标是按层识别并保留每层中间结果。

## 解题过程

AAEncode 使用颜文字形式的 JavaScript 表达式构造字符串和 `Function`。还原这一层后，函数体中的有效载荷为：

```text
4A5645475153435A4B345957595A4A524B4A3347454D4A5A4F5249554F4E4A564C415A453235323249524844533D3D3D
```

它只包含十六进制字符且长度为偶数，按 Base16 解码得到：

```text
JVEGQSCZK4YWYZJRKJ3GEMJZORIUONJVLAZE2522IRHDS===
```

该字符串符合 Base32 字符集，再解一层得到：

```text
MHhHYW1le1Rvb19tQG55X2MwZDN9
```

最后进行 Base64 解码。后三层可以完全在本地复现：

```python
import base64

s = b"4A5645475153435A4B345957595A4A524B4A3347454D4A5A4F5249554F4E4A564C415A453235323249524844533D3D3D"
s = base64.b16decode(s)
print(s.decode())
s = base64.b32decode(s)
print(s.decode())
s = base64.b64decode(s)
print(s.decode())
```

最终得到：

```text
0xGame{Too_m@ny_c0d3}
```

## 方法总结

多层编码应根据字符集和填充特征逐层判断，并记录中间结果：十六进制串对应 Base16，`A-Z2-7` 对应 Base32，常见字母数字与 `=` 填充对应 Base64。不要把未知 JavaScript 直接交给不可信在线页面执行。
