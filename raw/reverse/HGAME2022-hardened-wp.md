# hardened

## 题目简述

题目给出经过梆梆免费版加固的 APK。Java 层只负责加载 `libenc.so` 并调用 JNI 方法，真正的校验位于 native 库中；库内又使用 OLLVM 风格控制流和字符串混淆隐藏 AES 参数及变换后的 Base64 字母表。

## 解题过程

先识别加固类型并脱壳。无 root 环境可以使用 [BlackDex](https://github.com/CodingGay/BlackDex) 在运行时转储已加载的 DEX；有 root 环境也可以用相应的 Xposed 脱壳模块。脱壳后的 Java 代码会显示应用加载了 `libenc.so`，因此后续分析目标应转向该库，而不是继续停留在 APK 外壳。

在 `libenc.so` 中定位 JNI 导出函数与 AES 调用。OLLVM 使反编译结果不够直观，但查看 AES key、IV 和字母表缓冲区的交叉引用，仍能找到字符串解密后的内存地址。也可以用 Frida 在库基址上直接读取这些字符串：

```javascript
function printString(offset) {
    const base = Module.findBaseAddress("libenc.so");
    console.log(offset, base.add(offset).readCString());
}

printString(0x31070);
printString(0x31020);
printString(0x31050);
```

启动应用并加载脚本：

```bash
frida -U -f com.example.hardened -l script.js --no-pause
```

运行时输出恢复出三项关键数据：

```text
0x31070  0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz+/=
0x31020  JUST_A_NORMAL_KEY_FOR_YOU_TO_DEC
0x31050  you_find_me!!!!!
```

第一项说明程序在标准 Base64 之后又做了一次换表。标准字母表与程序字母表分别为：

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz+/
```

内置密文为：

```text
mXYxnHYp61u/5qksdDel6TgiKqcvUbBkX3xErlR4lO0aEAdU0acJY8PRSVXJxxsRR8Dq9MTJhkWLSbBvCG5gtm==
```

先把程序字母表翻译回标准 Base64，再进行 AES-CBC 解密和 PKCS#7 去填充：

```python
import base64

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
changed = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz+/"

key = b"JUST_A_NORMAL_KEY_FOR_YOU_TO_DEC"
iv = b"you_find_me!!!!!"
ciphertext = (
    "mXYxnHYp61u/5qksdDel6TgiKqcvUbBkX3xErlR4lO0aEAdU0acJY8PRSVXJxxsRR8"
    "Dq9MTJhkWLSbBvCG5gtm=="
)

normal_b64 = ciphertext.translate(str.maketrans(changed, standard))
encrypted = base64.b64decode(normal_b64)
plaintext = AES.new(key, AES.MODE_CBC, iv).decrypt(encrypted)
print(unpad(plaintext, AES.block_size).decode())
```

得到：

```text
hgame{cONGraTUl4T|0N5!N0w_yoU_C4n_eN?Oy~thE~MUsIc}
```

## 方法总结

这题虽然以 APK 为载体，决定性障碍却是 native 库脱壳后的逆向与 OLLVM 字符串恢复，因此归入 Reverse。处理这类题时应先穿透加固层，沿 JNI 调用定位真正算法；对于难以静态阅读的字符串混淆，运行时读取解密后的缓冲区通常比强行整理控制流更直接。最后还要注意 AES 外额外存在一次 Base64 换表，不能把密文直接交给标准解码器。
