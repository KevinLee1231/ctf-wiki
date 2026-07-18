# week1easyBase

## 题目简述

题目把明文连续经过 Base16 和 Base64 表示。Base 编码不提供保密性：Base64 字符串通常由字母、数字、`+`、`/` 和末尾的 `=` 组成；解开后若得到偶数长度且只含 `0-9a-fA-F` 的文本，即可继续按十六进制解码。

当前公开仓库没有收录这道题的原始密文，因此不能凭空补写特定输入；解码链和可直接复现的方法如下。

## 解题过程

先对题目字符串进行 Base64 解码，得到 Base16 文本，再将十六进制文本还原为字节：

```python
import base64

encoded = input("cipher: ").strip().encode()
hex_text = base64.b64decode(encoded, validate=True).decode()

if len(hex_text) % 2 != 0:
    raise ValueError("Base16 文本长度不是偶数")

plain = bytes.fromhex(hex_text)
print(plain.decode())
```

若第一层结果不满足十六进制特征，应重新检查解码顺序、复制时是否遗漏 `=` 填充，以及题面是否使用了 URL-safe Base64 字符表。

## 方法总结

- 核心技巧：根据每一层输出的字符集特征，依次执行 Base64 和 Base16 解码。
- 识别信号：外层符合 Base64 字符表，内层是偶数长度的十六进制字符串。
- 复用要点：编码层数应由中间结果验证，不要只凭题名盲目重复解码；缺失原始密文时应明确标注，不能伪造样例冒充题目数据。
