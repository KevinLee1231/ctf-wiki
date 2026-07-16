# FirstSight-Android

## 题目简述

附件是包名为 `com.example.ezandroid` 的 Android APK。`MainActivity` 会用应用内的 `B62encode` 类编码用户输入，再与字符串资源 `R.string.secret` 比较。由于目标串和编码算法都在客户端，直接提取资源并做 Base62 逆运算即可得到正确输入。

## 解题过程

使用 JADX 打开 APK，沿 `MainActivity` 的提交按钮逻辑可以看到：

```java
B62encode.encode(userInput).equals(getString(R.string.secret))
```

目标值不在 Java 常量中，而在 `res/values/strings.xml`。从资源表恢复出的完整字符串为：

```text
J6IwkwOvgGFthiofcwab0ka7KrOMpVbfROQ9Jh5C5YMqfyLLdMrNoj4YGVh
```

`B62encode` 使用的字母表顺序是数字、大写字母、小写字母：

```text
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
```

把目标串视作一个 62 进制大整数，再转回大端字节即可本地解码：

```python
encoded = "J6IwkwOvgGFthiofcwab0ka7KrOMpVbfROQ9Jh5C5YMqfyLLdMrNoj4YGVh"
alphabet = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

value = 0
for char in encoded:
    value = value * 62 + alphabet.index(char)

decoded = value.to_bytes((value.bit_length() + 7) // 8, "big")
print(decoded.decode())
```

输出为：

```text
0xGame{caff454e-2238-42aa-a75a-75e9f5f1f769}
```

将该字符串输入应用也会通过比较。

## 方法总结

移动端本地校验不能保护秘密：代码、资源和常量都会随 APK 下发。分析时不仅要看反编译出的 Java/Kotlin 代码，还要追踪 `R.string.*` 等资源引用；Base62 还必须确认字母表顺序，同一串字符在不同 alphabet 下会解出完全不同的字节。
