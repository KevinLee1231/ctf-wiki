# Computer Troubles

## 题目简述

题目文本中夹着一段以 `1h` 开头的长字符串。它不是单一编码，而是把多种常见表示层编码连续套在一起，需要按每层输出的字符特征逐层判断。

## 解题过程

截取以 `1h` 开头的整行。首层字符范围和尾部填充符合 Ascii85，解码后得到只含十六进制字符的文本；十六进制还原后出现 Base64 字符串；再做 Base64 和一次十六进制解码即可。

核心过程可以写成：

```python
import base64

layer0 = encoded_line.encode()
layer1 = base64.a85decode(layer0)
layer2 = bytes.fromhex(layer1.decode())
layer3 = base64.b64decode(layer2)
plain = bytes.fromhex(layer3.decode()).decode()
print(plain)
```

输出为：

```text
UMDCTF-{updat3_is_r3ady}
```

## 方法总结

多层编码不应靠盲目在线解码。每解一层就检查输出的字节类型、可打印比例、字符集和填充符，再决定下一层。Ascii85、hex 和 Base64 都是可逆表示，不是加密；保留原始字节并避免多余的字符串转义，能够减少层间损坏。
