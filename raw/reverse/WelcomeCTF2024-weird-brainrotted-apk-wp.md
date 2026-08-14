# Weird Brainrotted APK

## 题目简述

附件是 Android APK。应用把用户输入用硬编码密钥和 IV 执行 AES-CBC/PKCS7 加密，再与硬编码 Base64 密文比较。目标是反编译用户代码并直接解密目标密文。

## 解题过程

用 JADX 打开 APK，忽略 AndroidX、Glide 等库代码，定位包 `com.greyhats.skibidi.toilet` 下的 `Rizz` 类。决定性常量和算法为：

```java
KEY_STRING = "zsfuxwCqcUOfaXNhHxYvJfPIOEoPMiyL";
IV = "W644i2IVQjBBeth9";
RIZZ = "D7NQV/ledSLBd0zF11CPuPAz8y6D8kt/rQ4j5vNOWhFrlwjMsb40Hg4pEhoeVf3s";
Cipher.getInstance("AES/CBC/PKCS7Padding");
```

既然比较对象是输入的确定性加密结果，使用同一密钥与 IV 对 `RIZZ` 做 Base64 解码和 AES-CBC 解密即可：

```python
from base64 import b64decode
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b"zsfuxwCqcUOfaXNhHxYvJfPIOEoPMiyL"
iv = b"W644i2IVQjBBeth9"
ct = b64decode("D7NQV/ledSLBd0zF11CPuPAz8y6D8kt/rQ4j5vNOWhFrlwjMsb40Hg4pEhoeVf3s")
print(unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16))
```

结果为：

```text
grey{skibidi_toilet_W_level_500_gyatt_rizz}
```

APK 的 GIF 和启动图标不参与校验，没有必要作为 WP 图片保留。

## 方法总结

客户端内硬编码的对称密钥不能保密。反编译 Android 应用后，应优先沿按钮回调追踪到校验函数，提取算法、模式、填充、密钥、IV 和密文；只要这些参数齐全，就能在应用外复现逆运算。
