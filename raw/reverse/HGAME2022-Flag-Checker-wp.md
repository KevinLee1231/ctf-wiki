# Flag Checker

## 题目简述

附件是 Android APK。界面按钮读取用户输入，以固定密钥 `carol` 执行 RC4，再将结果 Base64 编码，与程序内置字符串比较。题目不依赖 Android 权限、组件或 IPC，决定性障碍是恢复普通 Java 加密逻辑，因此归入 Reverse。

## 解题过程

使用 JADX 或 JEB 定位 `MainActivity` 的按钮回调，可以整理出核心逻辑：

```java
SecretKeySpec key = new SecretKeySpec(
    "carol".getBytes(), 0, 5, "RC4"
);
Cipher cipher = Cipher.getInstance("RC4");
cipher.init(Cipher.ENCRYPT_MODE, key);
byte[] encrypted = cipher.doFinal(input.getBytes());

String result = Base64.encodeToString(encrypted, 0).replace("\n", "");
```

比较目标为：

```text
mg6CITV6GEaFDTYnObFmENOAVjKcQmGncF90WhqvCFyhhsyqq1s=
```

RC4 是对称流密码，加密和解密都使用同一密钥流。先做 Base64 解码，再用密钥 `carol` 执行 RC4 即可：

```python
from base64 import b64decode

from Crypto.Cipher import ARC4

ciphertext = b64decode(
    "mg6CITV6GEaFDTYnObFmENOAVjKcQmGncF90WhqvCFyhhsyqq1s="
)
plaintext = ARC4.new(b"carol").decrypt(ciphertext)
print(plaintext.decode())
```

结果为：

```text
hgame{weLC0ME_To-tHE_WORLD_oF-AnDr0|D}
```

## 方法总结

APK 只是承载形式，关键仍是跟踪输入到比较点的数据流。确认顺序为“RC4 后 Base64”后，逆向必须反过来执行“Base64 解码后 RC4”。固定硬编码密钥和比较字符串让整个过程可以完全离线复现。
