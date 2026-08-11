# ezApk

## 题目简述

附件是去除符号的 Kotlin APK。按钮回调最终调用 AES-CBC 解密函数，但密文和口令没有直接以易读变量名出现，需要借助 JADX/JEB 的重命名与资源引用功能沿调用链定位。AES 密钥是 `SHA-256(secret)`，IV 是 `MD5(secret)`，密文先做 Base64 解码，再以 PKCS#7 去填充。

## 解题过程

用 JADX 或支持重命名的反编译器打开 APK，从界面上的 `Again?` 等可见字符串查找交叉引用，追到按钮监听器和 `MainActivity` 中的解密函数。函数逻辑可整理为：

```kotlin
val secretKey = "A_HIDDEN_KEY"
val key = SecretKeySpec(
    MessageDigest.getInstance("SHA-256").digest(secretKey.toByteArray()),
    "AES"
)
val iv = IvParameterSpec(
    MessageDigest.getInstance("MD5").digest(secretKey.toByteArray())
)
val cipher = Cipher.getInstance("AES/CBC/PKCS7Padding", "BC")
```

密文为：

```text
EEB23sI1Wd9Gvhvk1sgWyQZhjilnYwCi5au1guzOaIg5dMAj9qPA7lnIyVoPSdRY
```

Python 中可以直接复现相同派生和解密：

```python
import base64
import hashlib

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

secret = b"A_HIDDEN_KEY"
ciphertext = base64.b64decode(
    "EEB23sI1Wd9Gvhvk1sgWyQZhjilnYwCi5au1guzOaIg5dMAj9qPA7lnIyVoPSdRY"
)

key = hashlib.sha256(secret).digest()
iv = hashlib.md5(secret).digest()
plain = AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)
print(unpad(plain, AES.block_size).decode())
```

得到：

```text
hgame{jUst_A_3z4pp_write_in_k07l1n}
```

## 方法总结

APK 只是载体，本题的决定性障碍是恢复普通程序的数据流和 AES 参数，因此归入 Reverse。分析去符号 Kotlin 代码时，从资源字符串和 UI 回调反向追踪通常比从入口顺序阅读更快；还要区分“原始口令”和真正传给 AES 的哈希结果，IV 与 padding 任一项错误都会导致解密失败。
