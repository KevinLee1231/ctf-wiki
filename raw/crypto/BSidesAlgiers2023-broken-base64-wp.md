# Broken_Base64

## 题目简述

题目给出一段损坏的 Base64 文本：

```text
lbGxtYXRlc3tZMHVfaDRWM183MF91TmQzcjV0NG5EX0gwd19CNDUzNjRfdzBSazV9
```

同时明确 flag 格式为 `shellmates{...}`。关键并不是反复给末尾补 `=`，而是编码文本的开头被删掉了。Base64 每 3 个原始字节编码为 4 个字符，因此可以借助已知明文前缀恢复缺失的编码字符。

## 解题过程

完整 flag 的前三个字节必然是 `she`，其 Base64 编码为 `c2hl`。现有密文恰好从其中最后一个字符 `l` 开始，所以缺失的三个字符是 `c2h`：

```python
from base64 import b64decode, b64encode

broken = "lbGxtYXRlc3tZMHVfaDRWM183MF91TmQzcjV0NG5EX0gwd19CNDUzNjRfdzBSazV9"
missing = b64encode(b"she").decode()[:3]
fixed = missing + broken

flag = b64decode(fixed).decode()
print(flag)
```

运行结果为：

```text
shellmates{Y0u_h4V3_70_uNd3r5t4nD_H0w_B45364_w0Rk5}
```

仓库中的官方脚本会尝试在开头补若干个 `A`，这只能帮助确认数据确实发生了前缀错位，解出的内容仍可能带有错误字节。利用已知 flag 前缀恢复真实的 Base64 字符，才能得到无杂字节的结果。

## 方法总结

处理损坏的 Base64 时，应先区分缺失字符、错误填充和字符偏移。若已知明文格式，可以把已知前缀重新编码，比较其与现有字符串的重叠部分，从而恢复被删除的 Base64 字符。本题最终 flag 为 `shellmates{Y0u_h4V3_70_uNd3r5t4nD_H0w_B45364_w0Rk5}`。
