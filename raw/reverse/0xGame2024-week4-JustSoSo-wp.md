# JustSoSo

## 题目简述

附件是 Android APK。Java 层读取用户输入，用自定义 `ReversC4` 加密后进行 Base64 编码，再与资源 `R.string.encryptedFLAG` 比较；加密密钥由 JNI 方法 `getKey()` 从 `libsecret.so` 返回。

核心是同时恢复 native 密钥生成逻辑和 Java 层修改过的 RC4。这里的 S 盒不是标准 RC4 的 `0..255`，也不是简单的 `255..0`，而是精确初始化为 `256-i`，即 `256,255,...,1`。

## 解题过程

JADX 中可以看到 `MainActivity` 加载 `secret` 动态库并声明 native 方法：

```java
public static native int[] getKey();

static {
    System.loadLibrary("secret");
}
```

点击校验按钮后的数据流为：

```java
byte[] encrypted = ReversC4.encrypt(flagInput.getBytes(), getKey());
String result = Base64.encodeToString(encrypted, 0);
result.compareTo(getString(R.string.encryptedFLAG));
```

资源 `encryptedFLAG` 的内容为：

```text
nB9RCjwReif5P1H1MYO6m/hucCGjI6EE9wWEx/E4N+bO5k5ior6MnqAGQfc=
```

APK 同时包含 ARM、x86 和 x86_64 版本的 `libsecret.so`。分析符号

```text
Java_com_ctf_justsoso_MainActivity_getKey
```

可找到原始字符串 `just_0xGame_so`。SSE 伪代码只是一次并行处理 4 个字节，逐元素等价于：

```python
source = b"just_0xGame_so"
key = [(value * 2) ^ 0x7F for value in source]
```

得到的 14 个整数为：

```text
171, 149, 153, 151, 193, 31, 143, 241, 189, 165, 181, 193, 153, 161
```

`ReversC4.initial()` 的关键差异是：

```java
for (int i = 0; i < 256; i++) {
    s[i] = 256 - i;
    repeatedKey[i] = key[i % key.length];
}

int j = 0;
for (int i = 0; i < 256; i++) {
    j = (j + s[i] + repeatedKey[i]) % 256;
    int tmp = s[j];
    s[j] = s[i];
    s[i] = tmp;
}
```

后续 PRGA 与 RC4 结构相同，输出时把整数转换成 Java `byte`。由于流密码加解密相同，直接对 Base64 解码后的密文再次执行该算法：

```python
import base64


SOURCE = b"just_0xGame_so"
TARGET = "nB9RCjwReif5P1H1MYO6m/hucCGjI6EE9wWEx/E4N+bO5k5ior6MnqAGQfc="


def reverse_c4(data, key):
    s = [256 - i for i in range(256)]

    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) % 256
        s[i], s[j] = s[j], s[i]

    i = 0
    j = 0
    output = bytearray()
    for value in data:
        i = (i + 1) % 256
        j = (j + s[i]) % 256
        s[i], s[j] = s[j], s[i]
        stream = s[(s[i] + s[j]) % 256] & 0xFF
        output.append(value ^ stream)
    return bytes(output)


key = [(value * 2) ^ 0x7F for value in SOURCE]
ciphertext = base64.b64decode(TARGET)
print(reverse_c4(ciphertext, key).decode())
```

输出为：

```text
0xGame{fd51ce4b-4556-4cf9-9430-67480614e43b}
```

## 方法总结

Android 逆向遇到 `native` 方法时，需要沿 `System.loadLibrary` 找到各 ABI 下的 `.so`，按 JNI 导出名定位实现，再把 native 结果带回 Java 数据流。本题最容易写错的是 S 盒初值：源码使用 `256-i`，首项 256 会一直以整数参与置换，只有生成密钥流字节时才截断，不能擅自改成标准 RC4 的 `0..255` 或 `255..0`。
