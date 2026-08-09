# 尊嘟假嘟

## 题目简述

附件是一个 Android 应用。主要逻辑不在 `MainActivity`，而是在两个 Fragment、Navigation 参数、被重写的 `Toast`、动态加载的加密 DEX 与 JNI 校验函数之间串联。用户每次点击两个页面中的图片之一，会向参数 `zunjia` 追加 `0.o` 或 `o.0`；长度最多 36 字节，即最多 12 次选择。该字符串经“按位置异或 + 换表 Base64”编码后作为 RC4 密钥，用于解密内置密文。

解法依赖 Android Fragment 参数传递、资源中的加密 DEX、`DexClassLoader` 和 JNI 动态库，归入 `mobile`。

## 解题过程

### 跟踪 Fragment 与动态 DEX

Navigation 图为两个 Fragment 都注册了字符串参数 `zunjia`。点击事件读取当前参数并追加一个三字符片段：一个 Fragment 追加 `0.o`，另一个追加 `o.0`；达到 36 字节后不再继续拼接。

拼接结果传给自定义 `toast` 类。该类覆盖 `setText`，除显示文本外，还会调用 `DexCall.callDexMethod`，把当前字符串作为参数交给动态加载类的 `encode` 方法，然后进入 native `check`。

`callDexMethod` 的关键流程是：

```java
File dexDir = new File(context.getCacheDir(), "dex");
File dexFile = DexCall.copyDexFromAssets(context, dexFileName, dexDir);
DexClassLoader loader = new DexClassLoader(
    dexFile.getAbsolutePath(),
    dexDir.getAbsolutePath(),
    null,
    context.getClassLoader()
);
Object instance = loader.loadClass(className).getConstructor().newInstance();
Object result = instance.getClass()
        .getMethod(methodName, input.getClass())
        .invoke(instance, input);
dexFile.delete();
```

assets 中的 DEX 不是明文。JNI 的 `copyDexFromAssets` 中可见 IDEA 的乘法模数特征，解密后得到两个同构的 `encode` 方法。每个方法先执行：

```text
temp[i] = input[i] XOR i
```

再以自定义字母表进行 Base64 分组编码。真正参与索引的是下列字符串的前 64 个字符；末尾字符 `5` 位于索引 64，不会被 6 位索引访问：

```text
3GHIJKLMNOPQRSTUb=cdefghijklmnopWXYZ/12+406789VaqrstuvwxyzABCDEF5
```

### 还原 native 校验并枚举点击序列

native `check` 的决定性逻辑是标准 RC4：把 `encode(zunjia)` 作为密钥，解密 43 字节常量。由于原始字符串只有 `0.o`、`o.0` 两种三字节块，且最多 12 块，只需枚举

$$
\sum_{n=1}^{12}2^n=2^{13}-2=8190
$$

种候选，并用明文前缀 `hgame` 验证。完整 Python 求解脚本如下：

```python
CUSTOM_ALPHABET = (
    "3GHIJKLMNOPQRSTUb=cdefghijklmnopWXYZ/12+406789VaqrstuvwxyzABCDEF5"
)

CIPHERTEXT = bytes([
    0x7A, 0xC7, 0xC7, 0x94, 0x51, 0x82, 0xF5, 0x99,
    0x0C, 0x30, 0xC8, 0xCD, 0x97, 0xFE, 0x3D, 0xD2,
    0xAE, 0x0E, 0xBA, 0x83, 0x59, 0x87, 0xBB, 0xC6,
    0x35, 0xE1, 0x8C, 0x59, 0xEF, 0xAD, 0xFA, 0x94,
    0x74, 0xD3, 0x42, 0x27, 0x98, 0x77, 0x54, 0x3B,
    0x46, 0x5E, 0x95,
])


def custom_encode(raw: bytes) -> bytes:
    transformed = bytes(value ^ i for i, value in enumerate(raw))
    out: list[str] = []
    for offset in range(0, len(transformed), 3):
        group = int.from_bytes(
            transformed[offset:offset + 3].ljust(3, b"\x00"),
            "big",
        )
        out.extend([
            CUSTOM_ALPHABET[(group >> 18) & 0x3F],
            CUSTOM_ALPHABET[(group >> 12) & 0x3F],
            CUSTOM_ALPHABET[(group >> 6) & 0x3F],
            CUSTOM_ALPHABET[group & 0x3F],
        ])
    return "".join(out).encode()


def rc4(data: bytes, key: bytes) -> bytes:
    s = list(range(256))
    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) & 0xFF
        s[i], s[j] = s[j], s[i]

    out = bytearray(data)
    i = j = 0
    for pos in range(len(out)):
        i = (i + 1) & 0xFF
        j = (j + s[i]) & 0xFF
        s[i], s[j] = s[j], s[i]
        out[pos] ^= s[(s[i] + s[j]) & 0xFF]
    return bytes(out)


for length in range(1, 13):
    for value in range(1 << length):
        bits = f"{value:0{length}b}"
        clicks = "".join("0.o" if bit == "0" else "o.0" for bit in bits)
        plain = rc4(CIPHERTEXT, custom_encode(clicks.encode()))
        if plain.startswith(b"hgame"):
            print("clicks:", clicks)
            print("encoded key:", custom_encode(clicks.encode()).decode())
            print("flag:", plain.decode())
```

唯一命中的点击序列及结果为：

```text
clicks: o.00.oo.00.oo.0o.00.o0.o0.o0.o0.oo.0
encoded key: lsCsRs06kc/yTc=/isREiIyXNZvBOdXyPInvPtOsQZKUdWqd
flag: hgame{4af153b9-ed3e-420b-978c-eeff72318b49}
```

## 方法总结

- 核心技巧：从 Fragment 参数追踪 UI 状态，穿过自定义 `Toast`、动态 DEX 和 JNI 边界，最终把有限点击序列化为可枚举的 RC4 密钥空间。
- 识别信号：APK 中出现 `DexClassLoader`、assets 内不可直接解析的 DEX、`copyDexFromAssets` native 方法和覆盖系统控件的同名类时，应沿 Java/JNI/动态字节码三层继续追踪，而不是只看 `MainActivity`。
- 复用要点：先把每层转换写成独立函数，再用已知 flag 前缀作为验证 oracle。长度 36 指的是由 12 个三字节块组成的原始参数，不是编码后的 RC4 密钥长度。
