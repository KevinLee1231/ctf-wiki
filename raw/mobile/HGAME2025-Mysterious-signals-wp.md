# Mysterious signals

## 题目简述

附件由 Android 客户端和配套服务端组成。客户端把用户名、文件名和端口传给 `com.nobody.andsign.SSSign.a`，本地 so 负责生成请求签名，服务端则读取指定文件并返回一段加密后的消息。

逆向服务端可发现 flag 文件名为 `h1g1a1m1e1`。因此有两条完整解法：

1. 在 Android 端 Hook `SSSign.a`，把原本的文件名 `hello` 改为 `h1g1a1m1e1`，让客户端自行生成合法签名并请求目标文件。
2. 逆向客户端与服务端的魔改 TEA，离线解密服务端返回的密文。

## 解题过程

反编译 `MainActivity` 后，可以看到按钮事件调用：

```java
new SSSign().a("admin", "hello", port);
```

`SSSign.a` 会构造如下请求：

```http
POST /flag HTTP/1.1
sign: <native b(username + filename) 的返回值>
Content-Type: application/json

{"username":"admin","filename":"hello"}
```

其中 native 方法 `c` 负责解密字段名，实际得到 `username`、`filename` 和 `sign`；native 方法 `b` 负责计算签名。服务端的文件读取逻辑表明目标文件名为 `h1g1a1m1e1`。

第一种方法是直接 Hook Java 层参数：

```javascript
function hook() {
  const SSSign = Java.use("com.nobody.andsign.SSSign");

  SSSign["a"].implementation = function (username, filename, port) {
    console.log(
      `SSSign.a: username=${username}, filename=${filename}, port=${port}`
    );

    filename = "h1g1a1m1e1";
    const result = this["a"](username, filename, port);
    console.log(`result=${result}`);
    return result;
  };
}

Java.perform(hook);
```

native 库会扫描 `/proc/self/maps` 中的 `frida` 或 `LIBFRIDA` 特征；命中后会把种子从 `0x11223344` 改为 `0x44331122`，使签名失效。因此应使用去特征的 Frida Server，或在受控环境中修补这段检测。Hook 后客户端会为 `admin + h1g1a1m1e1` 重新计算签名，并把目标文件内容发回。

第二种方法不依赖仍然在线的赛题服务。以客户端原始参数 `admin`、`hello` 计算得到的签名为：

```text
41dce78c58dacf99cbbc2f1c20135745
```

服务端响应中的十六进制密文为：

```text
4b181fd6f8b852a9e23a4a7776e5f690
5b71341af8f194a5db07d2902d265540
1322e1c9a1a90e0d8676809c08484762
```

加密流程不是标准 TEA：程序先按 AES S-box 逐字节代换，再用 `e7c10e42b7a68e14` 和种子 `0x11223344` 生成 64 个 32 位轮密钥，最后对每个 8 字节分组执行 32 轮魔改 TEA。下面的脚本完整复现逆过程：

```python
MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9

# PDF 中给出的 AES 逆 S-box，按索引直接完成逆代换。
INV_SBOX = bytes.fromhex(
    "52 09 6a d5 30 36 a5 38 bf 40 a3 9e 81 f3 d7 fb "
    "7c e3 39 82 9b 2f ff 87 34 8e 43 44 c4 de e9 cb "
    "54 7b 94 32 a6 c2 23 3d ee 4c 95 0b 42 fa c3 4e "
    "08 2e a1 66 28 d9 24 b2 76 5b a2 49 6d 8b d1 25 "
    "72 f8 f6 64 86 68 98 16 d4 a4 5c cc 5d 65 b6 92 "
    "6c 70 48 50 fd ed b9 da 5e 15 46 57 a7 8d 9d 84 "
    "90 d8 ab 00 8c bc d3 0a f7 e4 58 05 b8 b3 45 06 "
    "d0 2c 1e 8f ca 3f 0f 02 c1 af bd 03 01 13 8a 6b "
    "3a 91 11 41 4f 67 dc ea 97 f2 cf ce f0 b4 e6 73 "
    "96 ac 74 22 e7 ad 35 85 e2 f9 37 e8 1c 75 df 6e "
    "47 f1 1a 71 1d 29 c5 89 6f b7 62 0e aa 18 be 1b "
    "fc 56 3e 4b c6 d2 79 20 9a db c0 fe 78 cd 5a f4 "
    "1f dd a8 33 88 07 c7 31 b1 12 10 59 27 80 ec 5f "
    "60 51 7f a9 19 b5 4a 0d 2d e5 7a 9f 93 c9 9c ef "
    "a0 e0 3b 4d ae 2a f5 b0 c8 eb bb 3c 83 53 99 61 "
    "17 2b 04 7e ba 77 d6 26 e1 69 14 63 55 21 0c 7d"
)


def make_round_keys():
    seed = 0x11223344
    material = b"e7c10e42b7a68e14"
    box = []

    for i in range(4):
        for j in range(4):
            for k in range(16):
                seed_byte = (seed >> (8 * j)) & 0xFF
                box.append(((seed_byte ^ material[k]) + i * j * k) & 0xFF)

    return [
        int.from_bytes(bytes(box[i:i + 4]), "little")
        for i in range(0, 256, 4)
    ]


def decrypt_block(left, right, keys):
    total = (DELTA * 32) & MASK

    for round_index in range(31, -1, -1):
        total = (total - DELTA) & MASK

        right = (
            right
            - (
                left
                ^ ((keys[2 * round_index + 1] + total) & MASK)
                ^ (left >> 5)
                ^ ((left << 4) & MASK)
            )
        ) & MASK

        left = (
            left
            - (
                right
                ^ ((keys[2 * round_index] + total) & MASK)
                ^ (right >> 3)
                ^ ((right << 2) & MASK)
            )
        ) & MASK

    return left.to_bytes(4, "little") + right.to_bytes(4, "little")


ciphertext = bytes.fromhex(
    "4b181fd6f8b852a9e23a4a7776e5f690"
    "5b71341af8f194a5db07d2902d265540"
    "1322e1c9a1a90e0d8676809c08484762"
)

keys = make_round_keys()
substituted = bytearray()

for offset in range(0, len(ciphertext), 8):
    left = int.from_bytes(ciphertext[offset:offset + 4], "little")
    right = int.from_bytes(ciphertext[offset + 4:offset + 8], "little")
    substituted.extend(decrypt_block(left, right, keys))

plaintext = bytes(INV_SBOX[value] for value in substituted).rstrip(b"\x00")
print(plaintext.decode())
```

运行结果为：

```text
hgame{7be75491-2329-403b-9829-a8f042dd3ba0}
```

原 PDF 展示了解密结构但没有在正文中打印最终 flag；上述结果由密文离线复算，并与[公开选手复盘](https://astralprisma.github.io/2025/02/17/hgame_25/)交叉核对。

## 方法总结

期望解的核心是把客户端当作合法签名机：先从服务端确定真实文件名，再在 Java 方法入口替换参数，让后续 JSON、签名和请求保持一致。需要注意 native 库对 Frida 的特征检测会主动更换密钥种子。离线解则要严格复现“轮密钥生成、魔改 TEA、小端分组、AES S-box 逆代换”四层处理；只套标准 TEA 或忽略 S-box 都无法得到可读明文。由于决定性操作依赖 Android/JNI 与 Frida 运行时，本题归入 `mobile`，而不是仅按 native so 的存在归入普通 `reverse`。
