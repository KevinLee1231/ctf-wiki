# HackINI2024 not native

## 题目简述

题目附件是一个 APK，但应用使用 React Native 开发，关键校验逻辑不在普通 Java/Kotlin 类中，而是位于 `assets/index.android.bundle` 的 Hermes JavaScript 字节码。目标是识别 Hermes 载体、反编译 bundle，并逆转逐字符的 ASCII 偏移。

## 解题过程

APK 本质上是 ZIP 容器，解包后可以找到：

```text
assets/index.android.bundle
```

`file` 会将其识别为 Hermes JavaScript bytecode version 94。使用支持该版本的 Hermes 字节码反编译器恢复伪 JavaScript 后，可以在 `checkFlag` 附近看到目标数组：

```javascript
[114, 105, 100, 109, 107, 110, 96, 117, 100, 116, 122, 105,
 50, 115, 108, 52, 52, 96, 51, 54, 94, 53, 94, 111, 50, 88,
 94, 83, 50, 53, 98, 85, 94, 111, 51, 85, 48, 119, 50, 96,
 50, 79, 102, 50, 109, 52, 124]
```

校验器对输入逐字符映射：偶数下标的字符码减 1，奇数下标的字符码加 1，然后与该数组比较。逆运算因此是偶数下标加 1、奇数下标减 1：

```python
target = [
    114, 105, 100, 109, 107, 110, 96, 117, 100, 116, 122, 105,
    50, 115, 108, 52, 52, 96, 51, 54, 94, 53, 94, 111, 50, 88,
    94, 83, 50, 53, 98, 85, 94, 111, 51, 85, 48, 119, 50, 96,
    50, 79, 102, 50, 109, 52, 124,
]

flag = "".join(
    chr(value + 1 if index % 2 == 0 else value - 1)
    for index, value in enumerate(target)
)
print(flag)
```

输出为：

```text
shellmates{h3rm35_45_4_n3W_R34cT_n4T1v3_3Ng1n3}
```

## 方法总结

APK 分类不能只看扩展名。先检查框架特征，看到 `index.android.bundle` 后再用 `file` 判断是源码 bundle 还是 Hermes 字节码，可以避免把时间浪费在无关的 Java 包装层。恢复比较点后只需重建最小字符映射，并用正向校验逻辑重新计算数组确认结果。
