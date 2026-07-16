# 镜渊折枝

## 题目简述

题目附件是 Android APK。应用先用 Java/Kotlin 层的 AES-CTR 加密用户输入，再把 48 字节密文交给 native 层校验。native 校验并不是直接比较密文，而是先让第 $i$ 个字节与下标 $i$ 异或，再与内置目标数组比较。只要从 APK 中取出 AES 参数和 native 目标数组，就可以完全静态地逆推出 flag，无需实际运行应用。

## 解题过程

### 梳理 Java 与 native 调用链

反编译 `FlagChecker.check()` 可以看到完整流程：

1. 从字符串资源 `R.string.key` 读取 AES 密钥；
2. 将 `assets/secret.bin` 的 16 字节内容作为 IV；
3. 调用 `encrypt(plainText, key, iv)`；
4. 将加密结果传给 native 方法 `compare(length, data)`。

`FlagChecker.encrypt()` 明确使用 `AES/CTR/NoPadding`。APK 中的实际参数为：

```text
key = 0xgame-SecretKey
iv  = 8a a5 61 0c 74 41 1b 11 a3 53 68 de 56 bf 6a b3
```

因此，原 WP 中通过 Frida Hook `FlagChecker.encrypt()` 获取参数的方法是可行的，但不是必要条件；密钥和 IV 都能直接从资源中静态提取。

### 逆转 native 层比较

`libeasynative.so` 的 `compare()` 先要求数据长度为 48，再按两个字节一组执行校验。将反编译结果化简后，两条判断分别是：

```c
data[index - 1] ^ v8       == target[index - 1];
data[index]     ^ (v8 | 1) == target[index];
v8 += 2;
index += 2;
```

由于 `v8` 依次为偶数、`v8 | 1` 为紧随其后的奇数，整体等价于：

$$
\text{target}[i]=\text{ciphertext}[i]\oplus i,
\quad 0\le i<48
$$

所以 AES 密文就是 `target[i] ^ i`。目标数组位于 native 库的只读数据段：

```text
38 ea 9a 5a 3c fb 0c 98 1a 66 8b 46 6a 06 19 08
41 cb 36 9e 16 c5 be 2f f0 9d bf a7 34 67 51 4c
e5 c2 78 a5 0f 35 fc 1d 39 8b 49 38 34 69 30 f7
```

将下标异或和 AES-CTR 解密合并成一个脚本即可直接恢复 flag：

```python
from Crypto.Cipher import AES

target = bytes.fromhex(
    "38 ea 9a 5a 3c fb 0c 98 1a 66 8b 46 6a 06 19 08 "
    "41 cb 36 9e 16 c5 be 2f f0 9d bf a7 34 67 51 4c "
    "e5 c2 78 a5 0f 35 fc 1d 39 8b 49 38 34 69 30 f7"
)

key = b"0xgame-SecretKey"
iv = bytes.fromhex("8aa5610c74411b11a35368de56bf6ab3")

ciphertext = bytes(value ^ index for index, value in enumerate(target))
cipher = AES.new(
    key,
    AES.MODE_CTR,
    nonce=b"",
    initial_value=int.from_bytes(iv, "big"),
)
flag = cipher.decrypt(ciphertext).decode("utf-8")
print(flag)
```

脚本输出：

```text
0xGame{Native_1s_also_1nt3r3st1ng_ef492d80a257c}
```

## 方法总结

本题的校验链分为两层：Java/Kotlin 层负责 AES-CTR，加密参数藏在字符串资源和 asset 中；native 层负责长度检查和按下标异或后的数组比较。分析 Android native 题时，应先把跨层数据流串起来，再逐层求逆。本题中先执行 `target[i] ^ i` 得到密文，随后使用 APK 内的 key 和 IV 做 AES-CTR 解密，就能静态、完整地复现结果。
