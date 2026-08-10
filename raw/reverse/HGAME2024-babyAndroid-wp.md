# babyAndroid

## 题目简述

题目给出一个 APK，校验逻辑同时位于 Java 层和 Native 层：Java 层的 `check1` 是 RC4，Native 层的 `check2` 通过 JNI 动态注册映射到 `.so` 中的 AES 校验函数。题目附件存在出题失误：程序中准备了修改 S-box 的 AES，但附带密文实际由标准 AES 生成，所以最终应使用标准 AES-ECB 解密。

## 解题过程

### 分析 Java 层 RC4

用 JADX 打开 APK，可看到输入先经过 Java 层 `Check1.check`，通过后再调用 Native 方法 `check2`。`Check1` 是标准 RC4，但代码中出现的 `0x7f0f0030` 只是 Android 资源 ID，不是密钥本身。在 `res/values/strings.xml` 中找到真正的字符串：

```text
3e1fe1
```

RC4 密文换算为无符号字节后是：

```text
b5505030a84b672da559c45bca0506b8
```

Java 的 `byte` 是有符号的；反编译结果中如果出现负数，只需对其做 `value & 0xff`，例如 `-75` 就是 `0xb5`。用上述 key 执行 RC4 得到：

```text
G>IkH<aHu5FE3GSV
```

这 16 字节将作为后续 AES 密钥。

### 跟踪 JNI 动态注册

从 APK 中提取 `libbabyandroid.so`，载入 IDA 后先查看 `JNI_OnLoad`。将反编译器中的相关指针类型修正为 `JNIEnv *`后，可识别出如下逻辑：

```c
jclass activity = (*env)->FindClass(
    env,
    "com/feifei/babyandroid/MainActivity"
);
(*env)->RegisterNatives(env, activity, methods, 1);
```

`methods` 指向一个 `JNINativeMethod` 结构，保存 Java 方法名、方法签名和 Native 函数指针。由此可确认 Java 层的 `check2` 对应 Native 函数 `sub_B18`。

对 `sub_B18` 做常量、轮函数与密钥扩展特征分析，可识别为 AES。题目中还在 `.init_array` 执行的初始化函数里改写了 S-box，这原本会导致一个魔改 AES。但实际附件里的密文与标准 AES 相匹配，无需反向重建魔改 S-box。

### 标准 AES-ECB 解密

Native 层密文为：

```text
64a280fd1b20d28efc529e13eea1fd1e
660b7a72a31bd8366fdc3dee3c015763
```

使用 RC4 阶段得到的 16 字节 key，按 AES-ECB、无填充解密：

```python
from Crypto.Cipher import AES

key = b"G>IkH<aHu5FE3GSV"
ciphertext = bytes.fromhex(
    "64a280fd1b20d28efc529e13eea1fd1e"
    "660b7a72a31bd8366fdc3dee3c015763"
)

plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
print(plaintext.decode())
```

输出：

```text
hgame{df3972d1b09536096cc4dbc5c}
```

## 方法总结

- APK 中的校验链可能横跨 Java 与 Native；先在 JADX 中找调用关系，再从 `JNI_OnLoad`/`RegisterNatives` 恢复动态映射。
- Android 资源 ID 不是资源值，需回到 `res/values` 中查找；Java 有符号字节则要转为 $0\sim255$ 的无符号值。
- 识别 AES 不能只看函数名，要结合 S-box、轮常量、分组长度和密钥扩展。
- 当代码看似是魔改算法时，先用已知明密文或附件密文验证标准实现；本题的出题失误正好使标准 AES 直接有效。
