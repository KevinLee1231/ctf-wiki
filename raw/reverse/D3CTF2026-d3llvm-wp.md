# d3llvm

## 题目简述

这是一个 Android 游戏，提示为 `Enter the token and start playing the game.`。Java 层要求用户手写输入 64 个十六进制字符，MNN 模型负责识别手写轨迹，再把 token 交给 `FlagNative.nativeVerifyInput()` 验证；验证成功后进入小游戏，得分达到 100 时由 `FlagNative.nativeRevealFlag()` 显示 flag。

真正的难点不在游戏，而在三层 native 逻辑：

1. `libd3llvm_payload.so` 的 `.text` 在文件中被加密，必须先复现 `libd3llvm.so` 的 loader。
2. token 校验由大量 16 位非线性约束、8 条额外记录约束和一个 32 位滚动校验和组成。
3. flag 的 AES 密钥同时依赖正确 token 和认证成功后保存的 64 位运行状态。

最终结果为：

```text
token = 196f0d201332b47deb98221f33c7f4a13d03de6c2a77279c4dbc1f87e4d297a8
flag  = d3ctf{OLLVM_is_still_somewhat_useful_for_AI}
```

下文地址均为 ARM64 ELF 的 IDA 虚拟地址。

## 解题过程

### 1. APK 与 Java 层流程

解包 APK 后可以看到两组关键 native 库：

```text
lib/arm64-v8a/libd3llvm.so
lib/arm64-v8a/libd3llvm_payload.so
lib/arm64-v8a/libMNN.so
```

以及加密模型：

```text
assets/model/touch_model.mnn.enc
```

`FlagNative.smali` 声明了两个 native 方法：

```java
static native boolean nativeVerifyInput(String input);
static native String nativeRevealFlag();
```

`MainActivity` 只允许长度恰好为 `0x40` 的输入提交。每次手写识别都会调用 `MnnTouchClassifier.nativeRun()`；删除字符时则调用 `nativeTruncateInputState()`，说明分类器的 native handle 中还维护了一份与输入位置相关的状态。

`OpcodeDashGame` 的 Java 逻辑只在 `score >= 100` 时调用 flag 解密回调。它没有参与 token 或 AES 密钥的数学计算，因此静态求解时无需真正玩游戏。

### 2. 还原加密 payload

先在 IDA 中分析未加密的 `libd3llvm.so`。`JNI_OnLoad @ 0xfd7c` 的主流程为：

1. 获取 `JNIEnv`。
2. `dlopen("libd3llvm_payload.so")`。
3. 调用 `verify_and_load_encrypted_payload @ 0xfed0`。
4. 初始化 loader/payload bridge。
5. 解析并调用 payload 导出的 `Payload_OnLoad`。

`parse_and_verify_signinfo @ 0x1137c` 会查找 `D3SGN2` 元数据。题目中的实际字段为：

| 字段 | 值 |
|---|---|
| metadata 文件偏移 | `0x52b0` |
| magic | `D3SGN2\0\0` |
| version | `4` |
| flags | `1` |
| `.text` 虚拟地址 | `0x11c20` |
| `.text` 文件偏移 | `0x11c20` |
| `.text` 长度 | `0x2d7cc` |
| nonce | `34521f9758f487e22f4d06a268e084e6` |
| key | `01ddcad8108d9c27a95189a6a48da88e995243d1f4dd5b56223a81f2cf21e0da` |
| 密文 SHA-256 | `70b8abd0e8390ddd9ec529d0fb3859738af633ffbb0c9f8f154252e8bb366377` |
| 明文 SHA-256 | `ac4a4b6d774425ec553b3207607f4ab6f6052a72a893536e857eba8a91022f10` |

`xor_sha256_keystream @ 0x2bc28` 并不是标准流密码。它以 32 字节为一组，对第 `counter` 组计算：

```text
stream[counter] =
    SHA256(key || nonce || little_endian_u64(counter))

plaintext_block = ciphertext_block XOR stream[counter]
```

`decrypt_payload_text_and_verify @ 0x11828` 会暂时把 `.text` 改成 RWX，解密后检查明文哈希，再恢复为 RX。按相同算法静态解密 payload，或在运行时绕过完整性检查后 dump 内存，都可以得到能够正常反编译的真实代码。

### 3. 定位 JNI 核心函数

`register_payload_natives @ 0x258d8` 注册了以下方法：

| Java native 方法 | wrapper | 核心逻辑 |
|---|---:|---:|
| `nativeCreate` | `0x11cc0` | `0x22998` |
| `nativeRun` | `0x15808` | `0x26468` |
| `nativeTruncateInputState` | `0x15824` | `0x1c24c` |
| `nativeVerifyInput` | `0x15938` | `0x292c4` |
| `nativeRevealFlag` | `0x15954` | `0x2a444` |

`nativeCreate` 和 `nativeRun` 使用了控制流平坦化，直接阅读大函数效率很低。更有效的做法是先从 JNI 表、MNN 导入函数、静态数据表以及最终 flag 密文反向追踪，再分别还原小型辅助函数。

### 4. token 的 16 位约束

#### 4.1 输入表示

`parse_hex_token_words @ 0x34644` 要求输入长度严格等于 64，并把每 4 个十六进制字符按大端形式组成一个 `uint16_t`：

```text
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
└─w0─┘└─w1─┘ ...                                      └─w15┘
```

因此待求量为 `w[0] ... w[15]`，所有运算均在模 `2^16` 下进行。

#### 4.2 四个直接约束

程序保存了左右常数和旋转量，计算：

```text
w[index] = ROL16(left XOR right, rotation)
```

| `index` | left | right | rotation | 结果 |
|---:|---:|---:|---:|---:|
| 0 | `4d31` | `ae1c` | 3 | `196f` |
| 1 | `91a7` | `d1bd` | 7 | `0d20` |
| 4 | `2c5e` | `5f43` | 11 | `eb98` |
| 5 | `b083` | `4993` | 5 | `221f` |

#### 4.3 行、列和对角线混合

`mix4_token_words @ 0x34bc0` 的等价伪代码如下：

```python
first = a ^ rol16(d, 3)
left = rol16(first + 0x9e37, 5)
right = rol16(0x45d9 * (b + 0x101 * group), 9)
combined = rol16((left ^ right) + (c ^ 0xdaef), 3)
combined ^= 0x1337 * first
tail = rol16(0x27d4 * d + 0x33 * group, (group % 11) + 1)
result = combined ^ tail
```

checker 对 4×4 网格施加：

- 4 行：`group = 1,2,3,4`，目标为 `cead, 6d43, df57, d811`；
- 4 列：`group = 5,6,7,8`，目标为 `6558, a8dc, 35f7, 10e5`；
- 主对角线 `(w0,w5,w10,w15)`：`group = 11`，目标 `4434`；
- 副对角线 `(w3,w6,w9,w12)`：`group = 13`，目标 `0735`。

#### 4.4 八条 record 约束

第二个混合函数 `mix_record_words @ 0x34d68` 为：

```python
t0 = b ^ rol16(d, (group % 13) + 1)
left = rol16(a + 0x9e37, 5)
right = rol16(0x45d9 * t0, 9)
mixed = rol16((left ^ right) + (c ^ 0x7f4a), 3)
mixed ^= 0x1337 * a
tail = rol16(d ^ (0x1111 * group), (group % 7) + 1)
result = mixed + tail
```

实际的 8 条索引记录和目标如下：

| # | `a` | `b` | `c` | `d` | group | target |
|---:|---:|---:|---:|---:|---:|---:|
| 0 | 0 | 6 | 11 | 13 | 17 | `f826` |
| 1 | 4 | 10 | 15 | 1 | 19 | `d3ae` |
| 2 | 8 | 14 | 3 | 5 | 23 | `8ccc` |
| 3 | 12 | 2 | 7 | 9 | 29 | `0225` |
| 4 | 5 | 0 | 14 | 10 | 31 | `7c26` |
| 5 | 15 | 4 | 9 | 2 | 37 | `0982` |
| 6 | 6 | 12 | 1 | 11 | 41 | `ee37` |
| 7 | 3 | 8 | 13 | 7 | 43 | `a397` |

这里很容易漏掉后四条记录：目标表位于 `0x4ba8`，长度是 8，而不是只看第一组反编译分支时误以为的 4。

#### 4.5 最终 32 位校验和

`final_token_checksum @ 0x34f64` 使用排列：

```text
[7, 2, 13, 0, 11, 5, 15, 8, 1, 14, 4, 10, 6, 12, 3, 9]
```

初值为 `0x6d2b79f5`，每轮执行：

```python
mixed = rol16(w[perm[i]] + 0x1234 * i, (i % 13) + 1)
u = ((state ^ mixed) * 0x045d9f3b + 0x27100001) & 0xffffffff
state = rol32(u, 7) ^ (u >> 11)
```

最终必须等于 `0x53705512`。

#### 4.6 求解策略与 token

直接把所有非线性乘法和 32 位滚动校验和同时交给 Z3，求解会明显变慢。脚本采用：

1. 用直接约束固定 `w0,w1,w4,w5`。
2. 利用主对角线可以对第三个参数求逆的性质，从 `w15` 推出 `w10`。
3. 枚举 16 位 `w15`，再用 record #1 预筛，只剩
   `97a8` 和 `a29a` 两个候选。
4. Z3 只求解行、列、对角线和 8 条 record 约束。
5. 对模型候选具体计算最终 32 位校验和；不匹配就加入 blocking clause。

解得：

```text
w0..w15 =
196f 0d20 1332 b47d
eb98 221f 33c7 f4a1
3d03 de6c 2a77 279c
4dbc 1f87 e4d2 97a8
```

拼接得到：

```text
196f0d201332b47deb98221f33c7f4a13d03de6c2a77279c4dbc1f87e4d297a8
```

脚本会用纯整数版本重新检查每条约束，最终校验和确认为：

```text
0x53705512
```

### 5. 获取认证后的运行状态

token 校验通过后，`nativeRevealFlag` 会进入 `sub_2F7CC`。该函数的两个关键输入分别是 token 和全局变量 `qword_431C8`，随后由二者派生 AES 密钥。

`qword_431C8` 在 `nativeVerifyInput` 中初始化，并在认证成功时更新。与其继续逆向手写识别模型的全部运行状态，更直接的做法是在正确 token 通过验证后 hook `sub_2F7CC`，读取传入值：

```text
qword_431C8 = 0xa01c8100444fb480
```

这个值只有在认证流程完整执行后才有效，不能把程序启动时的初始随机值直接用于解密。

### 6. 派生 flag AES 密钥

token 验证通过时，程序会把 token 和 `qword_431C8` 保存到全局状态。`nativeRevealFlag` 检查认证标志后，将两者传给 `sub_2F7CC`。

首先对 token 的 ASCII 字节做 FNV-1a-64：

```text
token_hash = 0xc05f81ba6ac93cf5
```

`derive_flag_aes_key @ 0x2f664` 使用标准 SplitMix64：

```python
def splitmix64(x):
    x = (x + 0x9e3779b97f4a7c15) & MASK64
    x = ((x ^ (x >> 30)) * 0xbf58476d1ce4e5b9) & MASK64
    x = ((x ^ (x >> 27)) * 0x94d049bb133111eb) & MASK64
    return (x ^ (x >> 31)) & MASK64

seed = 0xa01c8100444fb480

k0 = splitmix64(
    token_hash ^ seed ^ 0xd3c7f19a5eed2026
)
k1 = splitmix64(
    seed ^ rol64(token_hash, 17) ^ 0xa11ce5c0dec0de42
)
key = little_endian_u64(k0) + little_endian_u64(k1)
```

具体结果为：

```text
k0  = 0xd2afe8c4612f0322
k1  = 0xaa8d52b4e3653bd4
key = 22032f61c4e8afd2d43b65e3b4528daa
```

payload 在 `0x4a94` 保存了 48 字节 AES 密文：

```text
f154eaeafaebf01674f062267087e584
f3842f342f59283dbf515aacf4cd01d1
51c2a502b36d45be5cb5f9b11942d2c1
```

使用上述 key 做 AES-128-ECB 解密并移除 PKCS#7 padding：

```text
64 33 63 74 66 7b 4f 4c 4c 56 4d 5f 69 73 5f 73
74 69 6c 6c 5f 73 6f 6d 65 77 68 61 74 5f 75 73
65 66 75 6c 5f 66 6f 72 5f 41 49 7d 04 04 04 04
```

去掉末尾 `04 04 04 04` 后得到：

```text
d3ctf{OLLVM_is_still_somewhat_useful_for_AI}
```

native 还会检查结果以 `d3ctf{` 开头、以 `}` 结尾。该检查也为 token、运行状态与密钥提供了独立验证。

### 7. 验证

把 token 求解、固定运行状态与 AES 解密整合后直接运行：

```bash
python solve.py
```

关键输出如下：

```text
[+] token checksum: 0x53705512
[+] login token:    196f0d201332b47deb98221f33c7f4a13d03de6c2a77279c4dbc1f87e4d297a8
[+] qword_431C8:    0xa01c8100444fb480
[+] flag AES key:   22032f61c4e8afd2d43b65e3b4528daa
[+] flag:           d3ctf{OLLVM_is_still_somewhat_useful_for_AI}
```

## 方法总结

- 核心技巧：先从未加密 loader 恢复 `.signinfo` 元数据与 SHA-256 流式异或算法，得到真实 payload；也可以在绕过完整性检查后直接 dump 解密代码。
- 约束求解：从 JNI 注册表定位 `nativeVerifyInput`，把 64 个十六进制字符还原为 16 个 16 位变量。先利用可逆混合关系缩小候选，再验证行列、对角线、8 条交叉约束和最终 32 位校验和。
- 状态获取：认证后的 `qword_431C8` 可以在 `sub_2F7CC` 入口直接读取，无需完整复现手写识别模型的运行状态。
- 最终解密：对 token 做 FNV-1a-64，再把 token hash 与 `qword_431C8` 输入 SplitMix64 派生 16 字节密钥，使用 AES-128-ECB 解密并校验 PKCS#7 与 `d3ctf{...}` 格式。
