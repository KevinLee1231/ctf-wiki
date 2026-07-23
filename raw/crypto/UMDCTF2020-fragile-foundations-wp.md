# Fragile Foundations

## 题目简述

题目名称中的 “Foundations” 暗示 Base 编码。生成逻辑把 flag 连续进行 42 次 Base64 编码，因此附件看起来是一段很长的可打印字符串，但没有额外的加密密钥或随机量。

## 解题过程

Base64 是可逆编码，每轮都可以用标准库解开。按照生成次数循环 42 轮：

```python
import base64
from pathlib import Path

data = Path("ciphertext").read_bytes().strip()
for _ in range(42):
    data = base64.b64decode(data)

print(data.decode())
```

最终得到：

```text
UMDCTF-{b@se64_15_my_f@v0r1t3_b@s3}
```

若不知道轮数，也可以在数据仍满足 Base64 字符集与填充规则时持续解码，并在出现 `UMDCTF-` 前缀后停止。

## 方法总结

重复编码不会增加密码学强度。处理多层 Base 编码时，应保留原始字节并逐层解码，避免中途按错误字符集转换；已知生成轮数时直接严格循环最可靠。
