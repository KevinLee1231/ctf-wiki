# SignIn

## 题目简述

密文只是两层标准编码：外层为 Base64，解码后得到只含 Base32 字符表与填充符的文本；再做一次 Base32 解码即可恢复 flag，不涉及密钥或密码分析。

## 解题过程

解码顺序是先处理外层 Base64，再把中间结果按 Base32 解码。使用本地 Python 可直接复现：

```python
from base64 import b32decode, b64decode

data = b"..."  # 题目给出的密文
middle = b64decode(data)
flag = b32decode(middle)
print(flag)
```

Base64 常见字符表包含大小写字母、数字、`+`、`/` 和末尾 `=`；外层解开后若主要由大写字母、数字 `2` 到 `7` 与 `=` 组成，就符合 Base32 的典型特征。

## 方法总结

编码题应逐层识别输入输出的字符集，并在每一步检查中间结果，而不是把 Base64、Base32 当作加密。正文中的标准库调用已经完整给出解码顺序，图形工具仅用于直观验证。
