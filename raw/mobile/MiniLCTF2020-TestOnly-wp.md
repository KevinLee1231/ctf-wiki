# MiniLCTF2020 - TestOnly?

## 题目简述

APK 的 Manifest 设置了 `android:testOnly="true"`，界面又启用 `FLAG_SECURE` 并用两个线程让 flag 短暂显示后清空。真正的 flag 生成逻辑仍在 Java 层：对固定长字符串做 SHA-1，再把摘要十六进制字符与 30 字节表逐位异或。

## 解题过程

测试包可用 ADB 的 `-t` 选项安装：

```sh
adb install -t TestOnly.apk
```

不过无需和快速闪烁的界面竞争。反编译 `MainActivity` 后可见 `MessageDigest.getInstance("SHA")`；在 Android/Java 中该别名对应 SHA-1。对固定字符串计算十六进制摘要：

```python
import hashlib

origin = b'B08020D0FACFDAF81DB46890E4040EDBB8613DA5ABF038F8B86BD44525D2E27B26E22ACD06388112D8467FD688C79CC7EA83F27440577350E8168C2560368616'
digest = hashlib.sha1(origin).hexdigest()
print(digest)
```

结果为：

```text
33d40461bb5dc676ac72cfdd51f68bc5f88668c7
```

再按源码执行异或：

```python
table = [85, 95, 5, 83, 75, 96, 94, 0, 17, 61,
         102, 87, 80, 123, 4, 105, 85, 83, 101, 109,
         55, 85, 23, 48, 106, 1, 40, 7, 97, 31]

plain = bytes(v ^ ord(digest[i]) for i, v in enumerate(table)).decode()
print(plain.replace('flag', 'minil'))
```

输出：

```text
minil{Th1s_S33M3_40R_T3sT_0N1Y}
```

源码注释里留有一个 32 位十六进制旧值，但实际 API 输出是 40 位 SHA-1；复现应以运行算法为准，而不是照抄注释。

## 方法总结

Android 的 `testOnly`、`FLAG_SECURE` 和快速刷新都只限制常规安装或观察，不会保护 DEX 中的算法与常量。遇到摘要算法别名时应实际计算并核对输出长度，避免把看起来像 MD5 的旧注释当成真实输入。
