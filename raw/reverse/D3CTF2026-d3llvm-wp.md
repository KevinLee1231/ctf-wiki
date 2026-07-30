# d3llvm

## 题目简述

题目给出一个 Android APK：

- 题目描述：`Enter the token and start playing the game.`
- 包名：`com.example.d3llvm`
- 主界面：`MainActivity`
- 游戏界面：`GameActivity`
- 原始附件：`app-debug.apk`
- APK SHA-256：`62a24710718576a839b58c71cb6a1a0392772d7d35ec4b864600014c6c9b7c87`

Java 层要求用户手写输入 64 个十六进制字符。输入通过 MNN 模型分类后，交给
`FlagNative.nativeVerifyInput()` 验证；验证成功才会进入小游戏。游戏得分达到
100 后，`OpcodeDashGame` 调用 `FlagNative.nativeRevealFlag()` 显示 flag。

真正的难点不在游戏，而在三层 native 逻辑：

1. `libd3llvm_payload.so` 的 `.text` 在文件中被加密，必须先复现
   `libd3llvm.so` 的 loader。
2. token 校验由大量 16 位非线性约束、8 条额外记录约束和一个 32 位滚动校验和组成。
3. flag 的 AES 密钥同时依赖正确 token 和 MNN 实际执行算子名称产生的状态种子。

最终结果为：

```text
token = 196f0d201332b47deb98221f33c7f4a13d03de6c2a77279c4dbc1f87e4d297a8
flag  = d3ctf{OLLVM_is_still_somewhat_useful_for_AI}
```

本文地址均为 ARM64 ELF 的 IDA 虚拟地址。已标注数据库保存在
`_analysis/d3llvm_loader.i64` 和 `_analysis/d3llvm_decrypted.i64`。

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

`MainActivity` 只允许长度恰好为 `0x40` 的输入提交。每次手写识别都会调用
`MnnTouchClassifier.nativeRun()`；删除字符时则调用
`nativeTruncateInputState()`，说明分类器的 native handle 中还维护了一份
与输入位置相关的状态。

`OpcodeDashGame` 的 Java 逻辑只在 `score >= 100` 时调用 flag 解密回调。它没有
参与 token 或 AES 密钥的数学计算，因此静态求解时无需真正玩游戏。

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

`xor_sha256_keystream @ 0x2bc28` 并不是标准流密码。它以 32 字节为一组，
对第 `counter` 组计算：

```text
stream[counter] =
    SHA256(key || nonce || little_endian_u64(counter))

plaintext_block = ciphertext_block XOR stream[counter]
```

`decrypt_payload_text_and_verify @ 0x11828` 会暂时把 `.text` 改成 RWX，解密后检查
明文哈希，再恢复为 RX。脚本对 APK 内原始 payload 做同样操作，输出：

```text
_analysis/libd3llvm_payload.decrypted.so
SHA-256 = c54f5afb2c2b2300d89fcb7869a5b884ac36a190105daed6cabeb5c0202343d9
```

重新把该文件载入 IDA 后，Hex-Rays 才能正常还原真实函数。

### 3. 定位 JNI 核心函数

`register_payload_natives @ 0x258d8` 注册了以下方法：

| Java native 方法 | wrapper | 核心逻辑 |
|---|---:|---:|
| `nativeCreate` | `0x11cc0` | `0x22998` |
| `nativeRun` | `0x15808` | `0x26468` |
| `nativeTruncateInputState` | `0x15824` | `0x1c24c` |
| `nativeVerifyInput` | `0x15938` | `0x292c4` |
| `nativeRevealFlag` | `0x15954` | `0x2a444` |

`nativeCreate` 和 `nativeRun` 使用了控制流平坦化，直接阅读大函数效率很低。更有效的
做法是先从 JNI 表、MNN 导入函数、静态数据表以及最终 flag 密文反向追踪，再分别还原
小型辅助函数。

### 4. token 的 16 位约束

#### 4.1 输入表示

`parse_hex_token_words @ 0x34644` 要求输入长度严格等于 64，并把每 4 个十六进制字符
按大端形式组成一个 `uint16_t`：

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

- 4 行：`group = 1,2,3,4`，目标为
  `cead, 6d43, df57, d811`；
- 4 列：`group = 5,6,7,8`，目标为
  `6558, a8dc, 35f7, 10e5`；
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

这里很容易漏掉后四条记录：目标表位于 `0x4ba8`，长度是 8，而不是只看第一组
反编译分支时误以为的 4。

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

### 5. 解密 MNN 模型

`decrypt_and_decode_mnn_model @ 0x3055c` 揭示模型容器并不是 3DES。字符串
`"This is 3DES"` 实际被补 4 个 NUL，作为 16 字节 AES-128 密钥：

```text
54 68 69 73 20 69 73 20 33 44 45 53 00 00 00 00
T  h  i  s     i  s     3  D  E  S \0 \0 \0 \0
```

解密流程是：

1. AES-128-ECB 解密；
2. 严格校验并移除 PKCS#7 padding；
3. 丢弃前 12 字节 `ln+yqvHkKzFM`；
4. Base64 解码剩余内容。

得到的原始 MNN FlatBuffer 为：

```text
_analysis/touch_model.raw.mnn
大小    = 80200 bytes
SHA-256 = d96a1ca465a33a513605dd3b409ea7f4ac4969dd262f4b8169671ad4e06e27b0
版本字符串 = 3.6.0
```

### 6. 从 MNN 执行计划恢复运行时种子

#### 6.1 随机种子是障眼法

`read_urandom_seed @ 0x16ae0` 确实会打开 `/dev/urandom` 并读取 8 字节，失败时才
使用时钟回退值。这看起来意味着 flag 密钥每次都不同，但继续追踪状态结构可见：

- `state + 0x18`：有效槽位数量；
- `state + 0x1c`：64 个槽位的有效标记；
- `state + 0x60`：64 个 `uint64_t` 值；
- `state + 0x260`：当前 64 位种子。

`store_operator_hash_for_input @ 0x181c8` 把一次推理得到的值写入当前
`row * 16 + col` 槽位。随后 `recompute_active_input_seed @ 0x1c0c0` 直接执行：

```python
seed = sum(value[i] for i in range(64) if active[i]) % (1 << 64)
```

所以 `/dev/urandom` 只影响尚未执行任何推理的初始状态。第一次有效推理后，它就会被
确定性结果覆盖。

#### 6.2 MNN after-callback

由 `nativeRun` 的 RTTI 可以定位两个 lambda。before-callback 恒返回 `true`；
after-callback 最终进入 `mnn_after_callback_accumulate_hash @ 0x1bd30`：

```python
accumulator += fnv1a64(operator_info.name())
```

`fnv1a64_cstring @ 0x2ded0` 是标准 FNV-1a-64：

```python
h = 0xcbf29ce484222325
for byte in name:
    h = ((h ^ byte) * 0x100000001b3) & 0xffffffffffffffff
```

MNN 的 callback 接口确实在每个实际执行的 op 前后传入 `OperatorInfo`，其中包含
运行时 op 名称。可参考
[MNN 官方 Session API](https://github.com/alibaba/MNN/wiki/session)。

不能直接把 FlatBuffer 中的 39 个节点名全部相加，因为 MNN 会做常量折叠、布局转换，
并生成额外 Raster command。使用与模型一致的官方 MNN 3.6.0 运行时进行
`runSessionWithCallBackInfo` 交叉验证，实际 after-callback 恰好出现 13 次：

| # | 运行时名称 | FNV-1a-64 |
|---:|---|---:|
| 0 | `getitem_raster_0` | `31978d089d3791e7` |
| 1 | `getitem` | `5adbc54f5b673890` |
| 2 | `getitem_3_raster_0` | `ee599fdc2c2ddd81` |
| 3 | `getitem_3` | `c2d5f9909c85564a` |
| 4 | `max_pool1d_raster_0` | `f110cfb2b8b6feb8` |
| 5 | `max_pool1d` | `295e951c73bc9f25` |
| 6 | `getitem_6_raster_0` | `63b54be227474bcc` |
| 7 | `getitem_6` | `c2d5f6909c855131` |
| 8 | `mean_raster_0` | `bdd828cb58f563c3` |
| 9 | `mean_raster_1` | `bdd827cb58f56210` |
| 10 | `logits__matmul_converted_raster_0` | `7410e844d20115cd` |
| 11 | `logits__matmul_converted` | `a09ad36295767fde` |
| 12 | `logits_raster_0` | `3386d2bf361caa38` |

这些哈希的模 `2^64` 和为：

```text
operator_sum = 0x4280720401113ed2
```

算子名称与具体手写浮点数据无关，因此每个字符槽位写入的都是同一个
`operator_sum`。正确 token 恰好填满 64 个槽位，于是：

```text
seed = 64 * operator_sum mod 2^64
     = 0xa01c8100444fb480
```

### 7. 派生 flag AES 密钥

token 验证通过时，`nativeVerifyInput_core` 会把 token 和上述最终 seed 保存到全局
状态。`nativeRevealFlag_core` 检查验证标志后，将两者传给
`decrypt_and_validate_flag @ 0x2f7cc`。

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

k0 = splitmix64(
    token_hash ^ seed ^ 0xd3c7f19a5eed2026
)
k1 = splitmix64(
    seed ^ ror64(token_hash, 47) ^ 0xa11ce5c0dec0de42
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

native 还会检查结果以 `d3ctf{` 开头、以 `}` 结尾。该检查也为密钥和 seed 提供了
独立的正确性验证。

### 8. 一键脚本与复现

`solve.py` 不需要 Android 模拟器，也不依赖 `_analysis/deps` 中用于交叉验证的 PyMNN。
运行时只需要 `ctf-tools` 环境已有的 `z3-solver` 和 `pycryptodome`：

```powershell
wsl bash -lc 'source /home/kali/miniforge3/etc/profile.d/conda.sh; conda activate ctf-tools; cd "/mnt/d/文档/新建文件夹/D3CTF2026/d3llvm"; python solve.py; conda deactivate'
```

关键输出如下：

```text
[+] plaintext SHA256: ac4a4b6d774425ec553b3207607f4ab6f6052a72a893536e857eba8a91022f10
[+] raw model SHA256: d96a1ca465a33a513605dd3b409ea7f4ac4969dd262f4b8169671ad4e06e27b0
[+] token checksum  : 0x53705512
[+] login token     : 196f0d201332b47deb98221f33c7f4a13d03de6c2a77279c4dbc1f87e4d297a8
[+] operator hashes : 0x4280720401113ed2
[+] runtime seed    : 0xa01c8100444fb480
[+] flag AES key    : 22032f61c4e8afd2d43b65e3b4528daa
[+] flag            : d3ctf{OLLVM_is_still_somewhat_useful_for_AI}
```

脚本同时生成：

- `_analysis/libd3llvm_payload.decrypted.so`
- `_analysis/touch_model.raw.mnn`
- `flag.txt`

## 方法总结

本题是一条完整的 Android loader、OLLVM、SMT 和 AI runtime 逆向链：

1. 先从未加密 loader 恢复 `D3SGN2` 元数据和 SHA-256 流式异或算法，获得可供
   IDA 分析的真实 payload。
2. 从 JNI 注册表定位验证和 reveal 核心，抽取 16 位 token 约束。先求结构约束，
   再具体计算滚动校验和，可避开 Z3 对大规模非线性位向量的性能问题。
3. `"This is 3DES"` 是误导信息，真实模型容器使用 AES-128-ECB、PKCS#7、12 字节
   前缀和 Base64。
4. `/dev/urandom` 也是障眼法：seed 会被 64 个输入槽位中的 MNN 算子哈希和覆盖。
5. 必须使用 MNN 的实际执行计划名称；直接哈希 FlatBuffer 节点会漏掉生成的 Raster
   command，也会错误包含已折叠节点。
6. 最后以正确 token 和确定性 MNN seed 派生 AES key，解出并通过 `d3ctf{...}`
   包装检查验证 flag。

交付文件：

- `solve.py`：端到端静态求解脚本；
- `d3llvm-wp.md`：本文；
- `flag.txt`：最终 flag；
- `_analysis/d3llvm_loader.i64`：带注释的 loader IDA 数据库；
- `_analysis/d3llvm_decrypted.i64`：带注释的解密 payload IDA 数据库。
