# SUCTF2026-flumel

## 题目简述

附件是一个 Flutter Android 应用。点击校验按钮后，界面先对输入执行 `trim()`，随后依次经过：

1. Dart 层构造固定密钥并执行自定义流密码 `Rc4Warp`；
2. Dart 通过 FFI 把 36 字节中间值和 `cache.snap.bundle` 一起传给 Native 函数 `qk9v`；
3. Native 校验并执行 Hermes bytecode bundle，再从 bundle 原始字节派生 AES-128-CBC 的运行时密钥与 IV；
4. Native 对中间值补 PKCS#7 padding、执行 AES-CBC，并与内置的 48 字节密文比较。

比赛期间附件曾被更换。最终附件只改动了三个 ABI 下的 `libjunk.so`，Flutter 代码、Hermes bundle 和其余主要文件没有变化；本文以仓库中的最终版 `flumel.apk` 为准。旧版 Native 还把输入和 bundle 混合成较重的约束，最终版已改为直接比较 AES 密文，因此不要继续沿用旧附件的求解模型。

## 解题过程

### 1. 定位 Flutter 校验链

解包 APK 后，关键文件包括：

```text
lib/arm64-v8a/libapp.so
lib/arm64-v8a/libjunk.so
assets/flutter_assets/bundles/cache.snap.bundle
```

题目使用 Dart VM 3.11.1。用 blutter 恢复 `libapp.so` 符号后，可以把 flag 相关逻辑定位到 `ctf_verifier.dart`。界面实际传入的是：

```dart
final candidate = _controller.text.trim();
final ok = await CtfVerifier.verify(candidate);
```

`CtfVerifier.verify` 的主线可以概括为：

```dart
final bundle = await _loadHermesBundle();
final encrypted = Rc4Warp(_buildRc4Key()).process(utf8.encode(candidate));
return _verifyInNativeAsync(encrypted, bundle);
```

Native 入口要求 `encrypted.length == 36`，所以最终 flag 去掉界面两侧空白后必须恰好是 36 字节。bundle 的 APK 内路径就是上面列出的 `assets/flutter_assets/bundles/cache.snap.bundle`。

### 2. 还原 `Rc4Warp`

`_buildRc4Key` 并非直接保存字符串，而是逐字节异或：

```python
def build_rc4_key():
    obf = [0x1f, 0x3b, 0x3f, 0x03, 0x00, 0x0a, 0xcf,
           0xe5, 0xe7, 0xe8, 0xca, 0xcc, 0xd2]
    return bytes(v ^ ((9 * i + 0x4b) & 0xff) for i, v in enumerate(obf))
```

解得：

```text
TobeorNottobe
```

`Rc4Warp` 保留了 RC4 的置换表思路，但 KSA、PRGA 和最终输出都加入了额外扰动。按 Dart 实现还原如下：

```python
def rotl8(v, s):
    return ((v << s) | (v >> (8 - s))) & 0xff


def rc4warp(key, data):
    s = list(range(256))
    j = 0
    salt = 0xc3

    for i in range(256):
        k0 = key[(5 * i + 1) % len(key)]
        k1 = key[(3 * i + 7) % len(key)]
        salt = rotl8(salt, 1)
        j = (j + s[i] + k0 + (k1 ^ salt) + i) & 0xff
        s[i], s[j] = s[j], s[i]

    i = j = 0
    twist = 0x9d
    out = bytearray()
    for value in data:
        i = (i + 1) & 0xff
        j = (j + s[i] + 11 * i) & 0xff
        s[i], s[j] = s[j], s[i]

        index = (s[i] + s[j] + (s[(i + j) & 0xff] ^ twist)) & 0xff
        stream = s[index]
        twist = rotl8(twist, 3)
        spice = s[(stream ^ twist) & 0xff]
        out.append(value ^ stream ^ spice ^ (13 * i & 0xff))

    return bytes(out)
```

每个输出字节都是输入字节与同一条确定性密钥流异或，因此该函数加密、解密相同：对结果再次执行 `rc4warp` 即可恢复原文。

### 3. 判断 Hermes 层是否参与最终验证

bundle 的魔数为 `c61fbc03`，bytecode 版本为 90。可使用 [hbctool](https://github.com/bongtrop/hbctool) 拆解 Hermes bytecode；该工具的用途是把 HBC 文件反汇编为 HASM、修改后再汇编，但其上游版本表没有直接覆盖 HBC90，需要采用兼容 HBC90 的补丁或 fork，并补齐对应 opcode/版本表，不能直接拿默认版本强行解析。

反汇编后会看到大量永远不执行的静态干扰代码。真正有意义的 JavaScript 会安装 `global.__j1`：它从固定 key 派生置换表和伪随机流，再对 16 字节输入做双重置换与链式异或。

但这里必须区分两个事实：

- `qk9v` 会检查 Hermes 文件、创建 runtime 并执行 bundle；执行失败会直接判错，所以 bundle 不能删除或随意替换；
- 最终版没有调用 `__j1` 的返回值参与 AES 运算。官方源码也明确说明，这条预先设计的加密支路在打包最终 APK 时没有接入验证链。

Android/Kotlin 层还存在从 bundle 派生 MethodChannel 名称的隐藏分支，它同样不是 Dart 最终校验所走的 FFI 主线。真正有用的是 bundle 的原始字节，因为 Native 会用它们派生 AES key 和 IV。

### 4. 还原 Native `qk9v`

`libjunk.so` 中的 `qk9v` 先检查输入长度和 bundle 完整性，并包含反调试、反 Frida 检查。静态求解不需要绕过这些动态检测，只要直接复现后半段算法即可。

Native 先对 bundle 计算 FNV-1a 与标准 CRC32，并执行三组 bundle 状态常量校验。通过完整性检查后，按下式派生运行时参数：

```text
key0 = b"youknowwhatImean"
iv0  = b"itsallintheflow!"

saltK = FNV1a32(bundle) & 0xff
saltI = (CRC32(bundle) >> 8) & 0xff

key[i] = key0[i]
       ^ bundle[(17*i + 11) % len(bundle)]
       ^ ((saltK + i) & 0xff)

iv[i]  = iv0[i]
       ^ bundle[(29*i + 7) % len(bundle)]
       ^ ((saltI + 3*i) & 0xff)
```

最终 APK 中的 bundle 长度为 `2501101` 字节，计算结果为：

```text
FNV-1a32 = 1f1663e3
CRC32    = 594baa34
AES key  = 9ae9908d89879e9981ca199e82cd1783
AES IV   = dcd9c3d2daca55dca4af2aafa63aa3e9
```

注意 IV 是完整的 16 字节，末字节 `e9` 不能遗漏。

Native 对 36 字节 `Rc4Warp` 结果补 12 个 `0x0c`，然后执行标准 AES-128-CBC。目标密文为：

```text
569670de6d7e270e7e27a189cec7082b
a1883f69796631adbd7c6d0fea9f281d
60f9d1277f1b007c36d631727753edcf
```

### 5. 逆向恢复 flag

逆过程是先用 bundle 派生参数解 AES-CBC、严格校验并去掉 PKCS#7 padding，再对 36 字节中间值执行一次 `Rc4Warp`。下面的脚本直接从最终版 APK 读取 bundle：

```python
import binascii
import zipfile

from Crypto.Cipher import AES


TARGET = bytes.fromhex(
    "569670de6d7e270e7e27a189cec7082b"
    "a1883f69796631adbd7c6d0fea9f281d"
    "60f9d1277f1b007c36d631727753edcf"
)


def fnv1a32(data):
    value = 0x811c9dc5
    for byte in data:
        value = ((value ^ byte) * 0x01000193) & 0xffffffff
    return value


def derive_key_iv(bundle):
    key0 = b"youknowwhatImean"
    iv0 = b"itsallintheflow!"
    salt_k = fnv1a32(bundle) & 0xff
    salt_i = (binascii.crc32(bundle) >> 8) & 0xff

    key = bytes(
        key0[i] ^ bundle[(17 * i + 11) % len(bundle)] ^ ((salt_k + i) & 0xff)
        for i in range(16)
    )
    iv = bytes(
        iv0[i] ^ bundle[(29 * i + 7) % len(bundle)] ^ ((salt_i + 3 * i) & 0xff)
        for i in range(16)
    )
    return key, iv


def unpad_pkcs7(data, block_size=16):
    if not data:
        raise ValueError("empty plaintext")
    pad = data[-1]
    if pad == 0 or pad > block_size or data[-pad:] != bytes([pad]) * pad:
        raise ValueError("invalid PKCS#7 padding")
    return data[:-pad]


# rotl8 与 rc4warp 使用上一节的实现。
with zipfile.ZipFile("flumel.apk") as apk:
    bundle = apk.read("assets/flutter_assets/bundles/cache.snap.bundle")

key, iv = derive_key_iv(bundle)
stage = unpad_pkcs7(AES.new(key, AES.MODE_CBC, iv).decrypt(TARGET))
flag = rc4warp(b"TobeorNottobe", stage)

print(key.hex())
print(iv.hex())
print(stage.hex())
print(flag.decode())
```

输出为：

```text
9ae9908d89879e9981ca199e82cd1783
dcd9c3d2daca55dca4af2aafa63aa3e9
2f3314c304c1fa86dbd85e331093d5959d7eae4bc2a903315194e53c9ca07babd8d8d743
SUCTF{w311_d0n3_y0u_kn0w_h3rm35_n0w}
```

因此 flag 为：

```text
SUCTF{w311_d0n3_y0u_kn0w_h3rm35_n0w}
```

## 方法总结

这题的关键不是把 Flutter、Kotlin、Hermes 和 Native 中出现的所有逻辑都复现出来，而是先还原真实数据流：输入先过 Dart `Rc4Warp`，Hermes bundle 必须能被 Native 校验和执行，但 `__j1` 的输出没有接入最终版验证；bundle 原始字节随后用于派生 AES key/IV，最后才执行 AES-CBC 密文比较。

处理多语言 Android 逆向时，应分别记录“代码是否被执行”“执行结果是否流向最终比较”“文件原始字节是否作为参数”这三个问题。这样既不会被未接线的 Hermes 算法和 MethodChannel 分支带偏，也不会误以为 Hermes 完全无关而遗漏 bundle 派生参数。附件发生过替换时，还应先固定最终样本并比较改动范围，否则很容易把旧版约束和新版密文校验混为一谈。
