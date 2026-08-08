# miniLCTF 2024 RustedRobot Writeup

## 题目简述

这是一道 Android 与 Rust JNI 混合逆向题。`MainActivity` 加载 `libmyrust.so`，把用户输入交给 `Java_com_doctor3_androidrusttest_MainActivity_invokeCheck`；native 层变换字符串后，再回调 Java `CryptoClass.encrypt(String[])` 完成 AES 校验。

公开仓库没有保留 APK 或 `libmyrust.so`，以下关键常量由官方题解和[公开赛后分析](https://blog.hxzzz.asia/archives/42/)交叉确认；AES 解密及最终字符串逆变换已在本地独立复现。

## 解题过程

### 还原 JNI 到 Java 的数据流

拆出 `libmyrust.so` 后定位 JNI 导出函数。Rust 代码把输入转为字符向量，对每个字符加 1，再反转得到第二个字符串；随后构造长度为 2 的 Java 字符串数组并调用静态方法 `CryptoClass.encrypt`。

Frida 可直接观察 native 层回调参数：

```javascript
Java.perform(function () {
    const CryptoClass = Java.use("com.doctor3.androidrusttest.CryptoClass");
    CryptoClass.encrypt.overload('[Ljava.lang.String;').implementation = function (a) {
        console.log("key = " + a[0]);
        console.log("transformed = " + a[1]);
        return this.encrypt(a);
    };
});
```

第一项固定为 AES key：

```text
btdfA2jeeljf.1bp
```

Java 端 `Cipher.getInstance("AES")` 在该环境中使用 AES-ECB/PKCS5Padding，并把加密结果与 32 字节常量比较。

### 解密常量并撤销字符串变换

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

signed = [
    49, -93, 51, -59, 24, -5, -59, 60,
    -45, -32, -55, -54, -89, 67, 42, -94,
    47, 110, 72, 13, 31, 55, 55, 34,
    127, 65, -120, 13, -109, -92, -71, -97,
]
ct = bytes(x % 256 for x in signed)
pt = unpad(AES.new(b"btdfA2jeeljf.1bp", AES.MODE_ECB).decrypt(ct), 16)
flag = "".join(chr(c - 1) for c in pt[::-1])
print(pt)
print(flag)
```

AES 明文是 native 变换后的字符串：

```text
~u1c1s`E4UTVS|GUDMjojn
```

反转并把每个字符减 1，得到：

```text
miniLCTF{RUST3D_r0b0t}
```

## 方法总结

混合应用不能只分析 Java 或 native 一侧。应先画清 `Java 输入 → JNI/Rust 变换 → Java AES 校验` 的跨语言数据流；动态 hook JNI 回调参数可以快速确认静态分析结论。最终脚本仍应从密文常量开始完整撤销 AES、反转和字符加一，而不是只记录 hook 到的明文。
