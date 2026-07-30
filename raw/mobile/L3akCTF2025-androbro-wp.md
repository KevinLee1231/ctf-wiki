# L3akCTF 2025 Androbro Writeup

## 题目简述

Androbro 是一道以 Android 运行时和 JNI 为核心的移动安全题。应用启动后会进行 root、模拟器检测，关键逻辑则藏在 `libragnar.so` 中。只有向运行时注册的广播接收器依次发送正确 action 和口令，应用才会解开 APK 资源中的隐藏 DEX；最终 flag 由该 DEX 内的 AES-CBC 代码解密得到。

题目界面的 root 提示只表达“检测到 root，应用拒绝运行”，没有额外视觉信息，因此将其转写为文字，不保留原解题文档中的界面截图。

## 解题过程

### 绕过启动检查

反编译 APK 后，`com.defensys.androbro.MainActivity` 的 `isRooted()` 会调用 `/system/bin/which su`，并结合设备信息执行模拟器检测。最直接的静态绕过方式是把对应 smali 方法固定为 `false`：

```smali
.method public static isRooted()Z
    .locals 1

    const/4 v0, 0x0
    return v0
.end method
```

重新打包、签名和安装后，应用可以正常进入主界面。也可以用 Frida hook Java 方法，目标相同：让前置检查不再提前终止 Activity。

### 还原动态广播与解锁口令

`MainActivity.onCreate()` 会加载 `libragnar.so` 并调用 native 初始化函数。静态分析该 so 可以看到，它动态注册了一个 `BroadcastReceiver`，接收两个 action：

```text
THE_TRIGER
THE_UNLOCKER
```

第一个 action 在题目中故意拼成 `TRIGER`，发送命令时必须保持原样。第二个 action 并未以明文完整保存，而是由两段等长字符串逐字节异或得到。

解锁口令由包名计算。设：

```python
p = b"com.defensys.androbro"
k = hashlib.sha256(p).hexdigest().encode()
unlock = ARC4.new(k).encrypt(p).hex()
```

这里的 RC4 密钥不是 SHA-256 的 32 个原始字节，而是其 64 字节十六进制 ASCII 表示。计算结果为：

```text
6a209693a9acaf10dcd2e425bab62a5e48698b7fc3
```

因此可依次发送广播：

```bash
adb shell am broadcast -a THE_TRIGER
adb shell am broadcast -a THE_UNLOCKER \
  --es key 6a209693a9acaf10dcd2e425bab62a5e48698b7fc3
```

随后点击应用中的检查按钮，native 逻辑便会处理隐藏载荷。

### 解开隐藏 DEX

APK 的 `assets/E/M/O/H/G/CMVASFLW.EXE` 并不是真正的 Windows 可执行文件。程序把上一步得到的 42 字节十六进制 ASCII 口令循环作为异或密钥；解密后的文件头为：

```text
dex\n035\x00
```

下面的脚本同时演示口令推导与 DEX 恢复：

```python
import hashlib
import zipfile
from Crypto.Cipher import ARC4

package_name = b"com.defensys.androbro"
rc4_key = hashlib.sha256(package_name).hexdigest().encode()
unlock_text = ARC4.new(rc4_key).encrypt(package_name).hex()
unlock_key = unlock_text.encode()

with zipfile.ZipFile("Androbro.apk") as apk:
    encrypted = apk.read("assets/E/M/O/H/G/CMVASFLW.EXE")

dex = bytes(
    value ^ unlock_key[index % len(unlock_key)]
    for index, value in enumerate(encrypted)
)
assert dex.startswith(b"dex\\n035\\x00")

with open("recovered.dex", "wb") as file:
    file.write(dex)
```

动态方案也可在触发广播后使用：

```bash
frida-dexdump -U -f com.defensys.androbro
```

两条路线恢复的是同一个 DEX。静态解法的优点是无需依赖注入时机，也能解释口令与隐藏资源之间的关系。

### 解密最终 flag

反编译恢复出的 `FlagChecker` 后，可以看到三个 Base64 常量：

```java
private static final String BASE64_ENCRYPTED_FLAG =
    "LbkzN+Zr+k6klBtEh0jWnGX6zjTPXXTCztliM8++ENqdkWdyT5nkPn3yQ2YCXh9oBpvd9ab7AKS2JJ2i5YBj+Q==";
private static final String BASE64_IV =
    "Z4drGE7JIeRhwjFxxw4kcA==";
private static final String BASE64_KEY =
    "QZrwuDw4+lFrKFRvznVl3A==";
```

`decryptFlag()` 对三者解码后使用 `AES/CBC/PKCS5Padding`。Python 中的等价恢复代码为：

```python
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

ciphertext = base64.b64decode(
    "LbkzN+Zr+k6klBtEh0jWnGX6zjTPXXTCztliM8++ENqdkWdyT5nkPn3yQ2YCXh9oBpvd9ab7AKS2JJ2i5YBj+Q=="
)
iv = base64.b64decode("Z4drGE7JIeRhwjFxxw4kcA==")
key = base64.b64decode("QZrwuDw4+lFrKFRvznVl3A==")

flag = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext), 16)
print(flag.decode())
```

输出为：

```text
L3AK{_Using_native_cpp__is_not_really_hard_xd_31412314}
```

## 方法总结

本题形成了“前置环境检测 → JNI 动态注册广播 → 包名派生 RC4 口令 → 循环异或隐藏 DEX → AES-CBC 解密”的完整链条。只修补 root 检测并不能直接得到 flag；还必须继续跟踪 native 注册的 Android 组件行为以及资源解密过程。

分析动态注册组件时，应重点检查 `JNI_OnLoad`、`RegisterNatives`、`registerReceiver`、`Intent.getAction()` 和 `getStringExtra()` 等调用。遇到看似可执行文件的 APK 资源，也应先验证解密后 magic，而不要根据扩展名判断文件类型。最后，推导过程中要严格区分“摘要原始字节”和“摘要的十六进制文本”，否则 RC4 口令会完全不同。
