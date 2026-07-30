# Looney Droids

## 题目简述

附件是 Android 应用 `rev2.apk`。界面并不是唯一入口：应用还导出了一个广播接收器，并把广播参数传给 JNI 本地函数。真正的 flag 解密函数只有在 Java 层反 root 检测、本地层反 Frida 检测以及一次性失败标记检查全部通过后才会注册。

## 解题过程

### 定位导出的广播入口

`AndroidManifest.xml` 中的 `RandomReceiver` 对外导出，并监听动作：

```text
com.tnemesis.rev2.B3ST_C4RTOON
```

`RandomReceiver.onReceive()` 读取名为 `cartoon` 的字符串额外参数，调用本地方法 `decodeMessage(cartoon)`，再把结果交给 `MainActivity` 显示。因此，能够触发解密的广播为：

```bash
adb shell am broadcast \
  -a com.tnemesis.rev2.B3ST_C4RTOON \
  --es cartoon "looney_droids" \
  -n com.tnemesis.rev2/.components.RandomReceiver
```

不过，直接发送广播通常不会得到 flag，因为 `decodeMessage` 的实现由 `JNI_OnLoad` 动态决定。

### 还原 JNI 注册和检测逻辑

Java 层的 `Application` 会反射执行 `Checker` 中以 `detect` 或 `check` 开头的公开检测方法。C++ 层还会检查 Frida 痕迹。`JNI_OnLoad` 的关键行为如下：

1. 如果 `/data/data/com.tnemesis.rev2/looney_droids.fails` 已存在，则提前返回，不注册本地解码函数。
2. 文件不存在时继续执行设备安全检查。
3. 安全检查通过，就把 Java 方法 `decodeMessage` 绑定到真正的 `decodeFlag`。
4. 检查失败，则绑定到只返回诱饵内容的实现。

这个失败标记会在首次运行过程中产生，因此仅重新启动应用不能恢复正确注册。动态解法可以用 Frida 拦截 libc 的 `access()`，仅对该路径伪造不存在；在 `librev2.so` 加载后，再把 `isDeviceSafe` 的返回值改为 `1`。核心逻辑如下：

```javascript
let checkingMarker = false;

Interceptor.attach(Module.findExportByName("libc.so", "access"), {
    onEnter(args) {
        const path = args[0].readUtf8String();
        checkingMarker =
            path === "/data/data/com.tnemesis.rev2/looney_droids.fails";
    },
    onLeave(retval) {
        if (checkingMarker) {
            retval.replace(-1);
        }
    }
});

Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"), {
    onEnter(args) {
        this.target = args[0].readCString().endsWith("librev2.so");
    },
    onLeave() {
        if (!this.target) {
            return;
        }
        const module = Process.getModuleByName("librev2.so");
        for (const symbol of module.enumerateExports()) {
            if (symbol.name.includes("isDeviceSafe")) {
                Interceptor.attach(symbol.address, {
                    onLeave(retval) {
                        retval.replace(1);
                    }
                });
            }
        }
    }
});
```

保存为 `solve.js`，先启动应用，再发送前述广播：

```bash
frida -U -f com.tnemesis.rev2 -l solve.js
adb shell am broadcast -a com.tnemesis.rev2.B3ST_C4RTOON --es cartoon "looney_droids" -n com.tnemesis.rev2/.components.RandomReceiver
```

### 离线复现解密

源码还能给出不依赖手机运行环境的验证路径。`Crypto.java` 使用 `AES/ECB/PKCS5Padding`，并在口令左侧补字符 `0` 直到 16 字节。题目名提示的口令 `looney_droids` 因而变为 `000looney_droids`。密文也固定在源码中，可以直接解密：

```python
from base64 import b64decode

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

password = "looney_droids"
key = password.rjust(16, "0").encode()
ciphertext = b64decode(
    "F8wvI61iS+4DkaYRVE8+vElfo/C9PqIS6E8ESCfE+Y8mnHmZqVXDhmFnpMrsPKMs"
)

plaintext = unpad(AES.new(key, AES.MODE_ECB).decrypt(ciphertext), 16)
print(plaintext.decode())
```

输出为：

```text
N0PS{l00n3y_t00ns_or_l00n3y_dr01ds}
```

## 方法总结

本题的决定性机制是 Android 组件暴露与 JNI 动态注册：广播接收器提供输入入口，`JNI_OnLoad` 再依据多层检测选择真实或诱饵实现。动态解法需要同时处理一次性标记和安全检查；源码审计则能进一步恢复 AES-ECB、补零规则、密文和口令，直接离线验证 flag。由于利用链依赖 Android 广播、应用生命周期和 JNI 注册行为，归入 Mobile 更合适。
