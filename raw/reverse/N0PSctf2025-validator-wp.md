# VALidATOR

## 题目简述

附件 `VALidATOR.apk` 只有一个整数输入框。Java 层把输入传给本地方法 `validateNumber(long)`；只有整数同时满足八个模方程时，本地函数才原样返回它，随后应用用该值解开 flag。核心障碍是从 ARM64 本地库中恢复约束，并用中国剩余定理求出唯一解。

## 解题过程

### 找到真正的校验函数

APK 的 Java 层声明了两个容易引起注意的 JNI 方法：

```java
public native long validateNumber(long n);
private static native long ConfusionAndDiffusion(long value);
```

提取 `lib/arm64-v8a/libvalib.so` 后分析 `Java_com_tnemesis_MainActivity_validateNumber`。编译器把取模运算优化成了乘法高位、移位和减法，反编译结果看起来比原始算式复杂，但逐组还原比较常量后可得到：

```text
n ≡    3 (mod    7)
n ≡   18 (mod   37)
n ≡   42 (mod   97)
n ≡  101 (mod  197)
n ≡  333 (mod  397)
n ≡  666 (mod  797)
n ≡ 1010 (mod 1597)
n ≡ 1337 (mod 3217)
```

八个模数两两互素，因此在所有模数乘积

```text
8045305309803704971
```

范围内存在唯一解。

### 用中国剩余定理求解

下面的脚本直接实现构造式 CRT，不依赖符号运算库：

```python
from math import prod

moduli = [7, 37, 97, 197, 397, 797, 1597, 3217]
residues = [3, 18, 42, 101, 333, 666, 1010, 1337]

M = prod(moduli)
x = 0

for residue, modulus in zip(residues, moduli):
    partial = M // modulus
    inverse = pow(partial, -1, modulus)
    x += residue * partial * inverse

x %= M
print(x)
print([x % modulus for modulus in moduli])
```

运行结果为：

```text
3869593450890186764
[3, 18, 42, 101, 333, 666, 1010, 1337]
```

余数列表与八条约束完全一致。将 `3869593450890186764` 输入应用，`validateNumber` 会原样返回该值；Java 层随后把它用于 AES-CBC 解密并显示：

```text
N0PS{7h3d4mndr01d15m3553dup}
```

### 排除两个干扰点

本地函数还包含架构判断：在 x86 版本上，`validateNumber` 无论收到什么输入都会返回 `0`。因此不能在常见的 x86 Android 模拟器里直接验证，应使用 ARM 设备、ARM 模拟环境，或静态求解后修改该分支。

`JNI_OnLoad` 会把 `ConfusionAndDiffusion(long)` 动态注册到另一个本地函数。该函数只休眠约一秒，然后原样返回参数，与校验和解密均无关，是刻意设置的干扰项。

## 方法总结

本题虽然以 Android APK 为载体，但得到 flag 的决定性障碍是恢复原生 ARM64 校验逻辑，所以归入 Reverse。面对编译器优化后的模运算，不必逐条还原所有乘法魔数：识别“计算余数并与常量比较”的重复结构，恢复八组模数和余数后即可交给 CRT。还需注意 x86 强制失败分支和无意义的 `ConfusionAndDiffusion`，避免把运行环境限制或延时函数误判为主线。
