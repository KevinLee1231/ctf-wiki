# Baby Android 2

## 题目简述

APK 的 Java 层从输入框读取字符串并调用 `FlagChecker.check`，但该方法以 `native` 声明，实际实现在 `libbabyandroid.so` 中。JNI 函数要求输入长度为 `0x17`，并对每个位置检查：

$$
input[i]=stored[(i^2)\bmod 0x2f].
$$

关键常量 `stored` 也直接存在 native 只读数据中。

## 解题过程

先用 JADX 确认 Java 到 JNI 的调用链：`MainActivity.onClick` 只负责取字符串和显示成功/失败提示，真正校验位于导出的 `Java_byuctf_babyandroid_FlagChecker_check`。

从反编译结果提取常量：

```text
bycnu)_aacGly~}tt+?=<_ML?f^i_vETkG+b{nDJrVp6=)=
```

只需复刻 23 次索引计算：

```python
stored = "bycnu)_aacGly~}tt+?=<_ML?f^i_vETkG+b{nDJrVp6=)="
flag = "".join(stored[(i * i) % 0x2f] for i in range(0x17))
print(flag)
```

输出为：

```text
byuctf{c++_in_an_apk??}
```

把该字符串输入应用后，Java 层收到 `true` 并显示成功提示，完成动态验证。

## 方法总结

- 核心技巧：沿 Java `native` 声明定位 JNI 实现，提取常量并复刻索引置换。
- 识别信号：APK 的关键方法没有 Java 方法体且通过 `System.loadLibrary` 加载 `.so` 时，应转向对应 ABI 的 native 库。
- 复用要点：先确认 JNI 字符串转换、长度限制、有符号性和取模常数，再写短脚本还原；无需反编译整个库。
