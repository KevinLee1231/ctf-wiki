# 旋转密码城

## 题目简述

题目附件是一个 Java JAR。程序把用户输入交给 `CaesarPlus` 处理，再与内置密文比较；目标是还原该字符变换并逆推出正确输入。

## 解题过程

反编译 `Main` 和 `CaesarPlus` 后，可以看到变换只作用于 ASCII 可打印字符 `!` 到 `~`，计算式为：

$$
c'=(c-33+47)\bmod 94+33
$$

这正是 ROT47。由于 $47\times2=94$，对同一字符串执行两次该变换会回到原文，因此加密函数本身也可用于解密。

程序中的比较密文为：

```text
_Iv2>6L424c_4c2\f__5\7fec\da32\3ef2`cgd4b46N
```

其中反斜杠是密文字符的一部分，Python 中应使用原始字符串，避免把 `\f` 当作转义序列：

```python
encoded = r"_Iv2>6L424c_4c2\f__5\7fec\da32\3ef2`cgd4b46N"

decoded = "".join(
    chr((ord(char) - 33 + 47) % 94 + 33)
    if 33 <= ord(char) <= 126 else char
    for char in encoded
)
print(decoded)
```

输出：

```text
0xGame{cac40c4a-700d-f764-52ba-b67a1485c3ce}
```

## 方法总结

本题需要从源码确认字符范围、位移量和取模基数，而不能只凭“凯撒密码”名称猜测算法。ROT47 在 94 个可打印字符上平移一半周期，具有自反性；同时要注意密文中的反斜杠在代码字符串里的转义问题。
