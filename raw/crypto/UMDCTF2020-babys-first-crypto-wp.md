# Baby's First Crypto

## 题目简述

题目给出一段经过逐字节变换的密文。生成代码对 flag 的每个字节执行加法并在 128 上取模：

```python
ciphertext = bytes((value + 13) % 128 for value in flag)
```

因此这不是需要密钥搜索的现代密码，而是定义在 7 位 ASCII 空间上的固定偏移。

## 解题过程

加密关系为

$$
c \equiv m + 13 \pmod {128}
$$

所以逐字节执行逆运算即可：

```python
from pathlib import Path

ciphertext = Path("ciphertext").read_bytes()
plaintext = bytes((value - 13) % 128 for value in ciphertext)
print(plaintext.decode())
```

输出为：

```text
UMDCTF-{1_1uv_crypt0}
```

## 方法总结

遇到短小的自定义“加密”时，应先从生成代码写出精确等式。这里的模数是 128，而不是常见的 256；直接在相同模空间内减去 13，就能无损恢复原文。
