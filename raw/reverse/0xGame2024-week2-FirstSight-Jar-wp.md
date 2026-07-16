# FirstSight-Jar

## 题目简述

附件是可执行 JAR。反编译 `EzJar.class` 后可见，程序对 UUID 中的十六进制字符执行模 16 仿射替换，其他字符原样保留，并将结果与固定密文比较。

## 解题过程

反编译得到核心算法：

```java
static String alphabet = "0123456789abcdef";

int x = alphabet.indexOf(character);
if (x < 0) {
    result.append(character);
} else {
    result.append(alphabet.charAt((x * 5 + 3) % 16));
}
```

密文为：

```text
ab50e920-4a97-70d1-b646-cdac5c873376
```

加密函数是：

$$
E(x)=5x+3\pmod{16}
$$

因为 $5^{-1}\equiv13\pmod{16}$，解密为：

$$
D(y)=13(y-3)\pmod{16}
$$

使用 Python 逐字符还原：

```python
alphabet = "0123456789abcdef"
ciphertext = "ab50e920-4a97-70d1-b646-cdac5c873376"
inverse = pow(5, -1, 16)

plaintext = "".join(
    alphabet[((alphabet.index(char) - 3) * inverse) % 16]
    if char in alphabet else char
    for char in ciphertext
)

print(plaintext)
print(f"0xGame{{{plaintext}}}")
```

输出：

```text
b8a9fe39-dbe4-4926-87d7-52b5a5140047
0xGame{b8a9fe39-dbe4-4926-87d7-52b5a5140047}
```

## 方法总结

JAR 中的 `.class` 通常能较完整地恢复为 Java 伪源码。确认字符表和仿射参数后，应使用模逆而不是普通除法；非十六进制的连字符不参与变换，按原位置保留。
