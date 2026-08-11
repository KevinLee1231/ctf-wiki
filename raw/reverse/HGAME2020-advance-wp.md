# advance

## 题目简述

程序实现了 Base64，但索引表不是标准字母表。只要从二进制中恢复自定义表，把密文字符的索引映射回标准 Base64 表，就能直接解码。

## 解题过程

程序中的自定义表为：

```text
abcdefghijklmnopqrstuvwxyz0123456789+/ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

标准 Base64 表为：

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

密文是：

```text
0g371wvVy9qPztz7xQ+PxNuKxQv74B/5n/zwuPfX
```

Base64 的每个字符只表示一个 6 位索引，因此无需重写整个编解码器：先按自定义表查出索引，再用标准表中同一索引的字符替换，最后交给标准库解码。

```python
import base64

custom = "abcdefghijklmnopqrstuvwxyz0123456789+/ABCDEFGHIJKLMNOPQRSTUVWXYZ"
standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
ciphertext = "0g371wvVy9qPztz7xQ+PxNuKxQv74B/5n/zwuPfX"

translated = ciphertext.translate(str.maketrans(custom, standard))
plaintext = base64.b64decode(translated)
print(plaintext.decode())
```

输出为：

```text
hgame{b45e6a_i5_50_eazy_6VVSQ}
```

## 方法总结

- 核心技巧：自定义 Base64 通常只改变“索引到字符”的映射，先换表再调用标准解码器即可。
- 识别信号：程序按每 3 字节拆成 4 组 6 位数据，并持有长度为 64 的字符数组。
- 复用要点：映射方向应是“密文所用自定义表 → 标准表”；方向写反会产生合法但错误的 Base64 字符串。
