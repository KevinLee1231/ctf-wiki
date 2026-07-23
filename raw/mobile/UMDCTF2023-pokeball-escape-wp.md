# Pokeball Escape

## 题目简述

题目给出一个 Android APK，启动后播放被困音效，并每 5 秒尝试一次“逃出精灵球”。提示中的 escape 不是退出应用，而是绕过应用对设备厂商属性的检查，让程序进入解密分支。

APK 同时包含 Kotlin/Smali 逻辑和 `libpokeballescape.so`。成功条件、AES 密钥生成器和加密图片之间的关系如下：

```text
ro.product.manufacturer == "Devon Corporation"
  -> 调用原生 getKey()
  -> AES-CBC 解密 res/raw/encrypted
  -> 显示逃脱图片
```

## 解题过程

### 还原设备属性检查

`Timer` 从应用启动时立即执行，此后每隔 `0x1388` 毫秒，也就是 5 秒调用一次 `MainActivity.newGame()`。该方法首先比较：

```text
systemInfo() == "Devon Corporation"
```

反汇编原生方法 `systemInfo()`，可以看到它把
`ro.product.manufacturer` 传给 `__system_property_get()`，再把结果转换成 Java 字符串。普通模拟器或手机的厂商属性自然不会是宝可梦世界中的 Devon Corporation，因此正常执行只会提示：

```text
!! CONDITIONS NOT MET TO ESCAPE !!
```

可用 Frida 在 Java 层替换 native 方法的返回值：

```javascript
Java.perform(function () {
    const MainActivity = Java.use(
        "com.example.pokeballescape.MainActivity"
    );

    MainActivity.systemInfo.implementation = function () {
        return "Devon Corporation";
    };
});
```

也可以修改 Smali，使判断直接进入成功分支。满足条件后，应用停止计时器和原音频，播放 `escape.mp3`，并调用原生 `getKey()` 解密图片。

### 静态恢复确定性密钥

不运行 APK 也能完成题目。`getKey()` 实际调用 `genRand(32)`；继续反汇编该函数可见：

```text
srand(1337)
alphabet = 0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
重复 32 次：alphabet[rand() % 62]
```

固定随机种子使所谓随机密钥完全可复现。按该流程生成：

```text
Tih5qqfWmdPbiXoLZAiEa6epQccUXnPQ
```

`Decrypt.smali` 则说明 `res/raw/encrypted` 的前 16 字节是 IV，剩余部分是
`AES/CBC/PKCS5Padding` 密文。离线解密代码如下：

```python
from pathlib import Path

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

data = Path("encrypted").read_bytes()
key = b"Tih5qqfWmdPbiXoLZAiEa6epQccUXnPQ"

plain = AES.new(key, AES.MODE_CBC, data[:16]).decrypt(data[16:])
Path("escaped-pokeball-flag.jpg").write_bytes(
    unpad(plain, AES.block_size)
)
```

解密结果保留了宝可梦场景与应用成功状态，具有独立的视觉信息：

![成功逃出精灵球后的场景，皮卡丘旁显示红色 flag](UMDCTF2023-pokeball-escape-wp/escaped-pokeball-flag.jpg)

最终得到：

```text
UMDCTF{c0ngrAtz_0N_th3_e5c@pe!}
```

## 方法总结

- 核心技巧：同时跟踪 Android 系统属性检查、JNI 方法和资源解密逻辑；仅关闭应用并不等于题目要求的 escape。
- 动态路线：Hook `systemInfo()` 或修补分支，让应用相信设备厂商是 `Devon Corporation`。
- 静态路线：识别 `srand(1337)` 带来的确定性，复现 32 字节密钥后直接解密资源。
- 安全结论：本地环境检查和固定种子生成的客户端密钥都不构成安全边界，攻击者既能修改返回值，也能离线恢复全部秘密。
