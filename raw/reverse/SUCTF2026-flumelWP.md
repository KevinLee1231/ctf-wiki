# SUCTF2026-flumel

## 题目简述
题目是 Flutter/Dart + native + Hermes bytecode 混合逆向。题干提示 `pay attention to the actual key and IV generation logic`，说明重点不只是找固定 AES 常量，而是确认 key/IV 的真实生成路径。输入先经 Flutter UI 读取、Dart `trim()` 和 UTF-8 编码，再经过自定义 `Rc4Warp` 流变换，随后与 APK 中的 `assets/flutter_assets/bundles/cache.snap.bundle` 一起传给 native。

新版 native `libjunk.so` 会验证并执行 Hermes bytecode，再基于 bundle 的真实字节派生 AES-128-CBC key/IV，对 36 字节输入加 padding 后生成 48 字节密文并与内置 target 比较。解题需要把 Dart、native 和 Hermes 三层逻辑串起来，不能只看 Java 层或旧版 placeholder。

## 解题过程
1. 结论

这题最终的完整校验链是：

1. Flutter UI 取用户输入字符串。

2. Dart 侧先 trim() ，再走 Utf8Encoder::convert 。

3. Dart 用自定义 Rc4Warp 对输入做 36 字节流变换，key 固定为 TobeorNottobe 。

4. Dart 从 APK 里读取 assets/flutter_assets/bundles/cache.snap.bundle 。

5. Dart 把 `Rc4Warp(flag_utf8_bytes)` 和 `cache.snap.bundle` 原始字节一起传给 `libjunk.so:qk9v`。

6. 新版 libjunk.so 会先验证并执行 Hermes bytecode，然后再基于 bundle 的真实字节生成
key / IV。

7. native 用这个 key / IV 对 36 字节输入做标准 AES-128-CBC，加上 0x0c * 12 padding 后得到
48 字节密文。

8. qk9v 直接把这 48 字节密文和内置 target 比较，相同则通过。

最终 flag：

```
SUCTF{w311_d0n3_y0u_kn0w_h3rm35_n0w}
```

1. 题目更新后先做 diff

出题人说题目有问题，重新给了新附件：

- 旧 APK: flumel.apk

- 新 APK: attachment/flumel.apk

先对比新旧 APK，结论非常关键：
- classes.dex 没变

- AndroidManifest.xml 没变
- assets/flutter_assets/bundles/cache.snap.bundle 没变
- libapp.so / libflutter.so / libhermes.so 没变

- 只有 libjunk.so 变了，而且三个架构都变了

实际 diff 结果：

```
('lib/arm64-v8a/libjunk.so', (45536, 2917375206), (50856, 4031669943))
('lib/armeabi-v7a/libjunk.so', (18100, 1126599256), (14516, 1084378020))
('lib/x86_64/libjunk.so', (48192, 1087723673), (51160, 2601474102))
```

这一步的意义是：

- Java 层不用重新开荒
- Dart 层不用重新开荒
- Hermes bundle 不用重新开荒

- 真正要重看的只有新的 libjunk.so

2. Java 层：没有变，但有个隐藏分支

Java 层虽然不是最终解题核心，但不能忽略，因为它一开始很容易把人带偏。

MainActivity 里有三段自定义逻辑：

- 把 data 统一转成 byte[]

- 计算一个 6 字节结果

- 从 bundle 字节流里动态解码字符串

结合 JADX MCP 和已有导出文件，可以确认 Java 在 onCreate() 时读取了：

```
assets/flutter_assets/bundles/cache.snap.bundle
```

然后从这份 bundle 字节流里动态解出一个隐藏 MethodChannel ：
- channel: zhbplw.dlfltnqsl
- methods:
  - `xspjrmbb`
  - `kiqlqwgh`
  - `nbwrpylw`
  - `emifxpoo`
  - `lchqtaqe`
  - `nzoagqgf`

handler 收到调用后，会要求参数里有：
- step
- slot
- state
- data

最后返回一个 6 字节结果。

这个分支说明两件事：

1. cache.snap.bundle 从一开始就不是普通资源文件。

2. 出题人确实把“bundle 内容参与校验”这个思路埋在了多个层里。

但在更新后的附件里，Java 这条线没有变化，也不是最终 flag 的主校验路径。

3. Dart 层：真正的主调用链

Flutter / Dart 这层才是主链入口。

通过 blutter 输出可以把调用链串起来：

```
_FlagCheckPageState::_verifyFlag
-> CtfVerifier::verify
-> _loadHermesBundle()
-> _buildRc4Key()
-> Utf8Encoder::convert(trimmed_input)
-> Rc4Warp::process(...)
-> _verifyInNativeAsync(...)
-> _nativeWorker
-> dlopen("libjunk.so")
-> lookup("qk9v")
-> qk9v(transformed_flag, 36, bundle_bytes, bundle_len)
```

这里要特别注意两点：

### 3.1 输入不是原始字符串，而是 UTF-8 字节

UI 输入经过 trim() 后，不是直接拿字符逐个参与计算，而是：

```
Utf8Encoder::convert
```

所以最终正确输入必须满足：

- 长度为 36 字节

- 是一个合法 UTF-8 可打印字符串

### 3.2 Dart 先做了一层自定义 Rc4Warp

Rc4Warp 不是标准 RC4，但本质仍然是“输入异或 keystream”的流加密，所以它是自逆的。

这里是题里第一个很容易还原错的地方。

我一开始少还原了一步 s[(j + k) & 0xff] 的参与，导致后续虽然能把 native 的 ciphertext 还
原出来，但逆不回真正 flag。

修正后的精确实现已经在脚本里，关键 PRGA 是：

```
j = (j + 1) & 0xFF
a = s[j]
k = (k + a + 11 * j) & 0xFF
c = s[k]
s[j], s[k] = s[k], s[j]
mix = s[(j + k) & 0xFF]
t = (s[j] + a + ((mix ^ seed) & 0xFF)) & 0xFF
d = s[t]
seed = rol8(seed, 3)
e = s[(d ^ seed) & 0xFF]
out[i] = in[i] ^ d ^ e ^ ((13 * j) & 0xFF)
```

固定 key 则来自 _buildRc4Key() ：

```
TobeorNottobe
```

4. Hermes：不是摆设，而且新版 libjunk.so 真的执行了它

cache.snap.bundle 不是任意二进制，而是标准 Hermes bytecode。

头部校验结果：

- magic: c61fbc03
- version: 90

这在脚本里也能直接读出来。

### 4.1 bundle 内部有什么

我把 Hermes bytecode 跑通后，确认里面有 6 个有语义的函数：
- global
- aa
- bb
- cc
- asa
- tbp

其中安装链是：

```
global (#0)
-> 调 installer closure #9002
-> #9002 创建 aa / bb / cc / asa / tbp
-> #9002 把 closure #9008 挂到 global.__j1
```

__j1 的行为很明确：

1. 要求参数长度为 16

2. 把 arg[i] & 0xff 复制到新数组

3. 调 tbp

tbp 又会：

1. 调 bb()

2. 调 cc(16, bb())

3. 取出 { sbox, stream }

4. 做 16 字节 block transform

5. 最后调 asa() 输出 hex

示例：

```
j1(bytes(range(16))) = d3594cc44ddc4695f93947d3a432078e
```

### 4.2 __pre / __post

在 Hermes 字符串表里还能看到：
- __j1
- __pre
- __post

但只有 __j1 有真实字节码引用。

__pre / __post 只是在字符串表里存在，没有实际调用点。

### 4.3 更新后的关键变化

旧版分析里，Hermes 更像是“bundle 字节参与混合”，但新附件里不是这样了。

新的 libjunk.so 里，qk9v 直接导入并调用了 Hermes 相关符号：
- HermesRuntime::isHermesBytecode
- makeHermesRuntime
- jsi::Value::~Value
- jsi::Buffer::~Buffer

并且在 qk9v 内部能确认有这条链：

1. 检查 bundle 是否是 Hermes bytecode

2. 初始化 runtime config

3. 创建 Hermes runtime

4. 构造 StaticBuffer

5. 用 "bundles/cache.snap.bundle" 作为源名执行这份 bundle

6. 返回值立刻析构

这说明：

- Hermes 已经直接进入主校验链

当 secret blob 哈希一下”• 不是单纯“把 bundle

不过还要注意一个细节：

- bundle 被执行了

- 但最终 key / IV 不是直接来自 __j1 的返回值

也就是说，Hermes 在这里更像是一个必须经过的 side-effect / 完整性阶段，而真正的 key / IV 仍然是
native 后面自己按 bundle 字节生成的。

5. 新 libjunk.so 的真正关键：提示说的就是 key / IV

出题人给的提示是：

```
Here's a hint: pay attention to the actual key and IV generation logic.
```

这个提示非常关键，因为它直接点破了最容易踩坑的点：

- AES 本身不是魔改重点

- 真正的坑在 key / IV 派生逻辑

### 5.1 anti-debug / anti-Frida 还在

新的 libjunk.so 仍然保留了：
- /proc/self/status + TracerPid
- /proc/self/maps
- /proc/self/task/*/comm
- frida
- frida-agent
- frida-gadget
- gum-js-loop
- linjector

这些都还在。

不过它们只影响动态调试，不影响静态还原算法。

### 5.2 真正的 key / IV 生成公式

最终确认下来的公式如下。

先对整个 cache.snap.bundle 做：
- FNV-1a 32
- CRC32 state

记：

- fnv32 = FNV1a32(bundle)
- crc_state = CRC32_state(bundle, seed=0xffffffff)
- crc_final = (~crc_state) & 0xffffffff

然后：

```
key[i] = bundle[(11 + 17 * i) % n] ^ ((fnv32 + i) & 0xff) ^ b"youknowwhatImean"
[i]
iv[i] = bundle[(7 + 29 * i) % n] ^ (((crc_final >> 8) + 3 * i) & 0xff) ^
b"itsallintheflow!"[i]
```

其中 n = len(bundle) 。

代入题目实际 bundle 后得到：

```
fnv32 = 0x1f1663e3
crc_state = 0xa6b455cb
crc_final = 0x594baa34
key = 9ae9908d89879e9981ca199e82cd1783
iv = dcd9c3d2daca55dca4af2aafa63aa3e9
```

这一步就是新附件真正修掉的地方。

旧版如果还沿用之前那套 bundle mixer / 伪 key / 伪 iv，会整条链都对不上。

7. 完整加密流程

到这里就可以把整个题目的前向加密流程完整写出来了。

### 7.1 输入阶段

用户输入：

```
flag_str
```

Dart 侧做：

```
trim(flag_str)
utf8_bytes = Utf8Encoder::convert(...)
```

要求最后是 36 字节。

### 7.2 Dart 侧自定义流变换

```
native_input = Rc4Warp(utf8_bytes, key="TobeorNottobe")
```

这一步输出 36 字节。

### 7.3 bundle 阶段

native 收到的另一个参数是：

```
bundle = assets/flutter_assets/bundles/cache.snap.bundle
```

然后 qk9v 会：

1. 确认它是合法 Hermes bytecode

2. 创建 Hermes runtime

3. 把 bundle 作为 "bundles/cache.snap.bundle" 执行一遍

### 7.4 key / IV 派生

执行完 Hermes 阶段后，native 继续对 bundle 原始字节做哈希并生成：

```
key = 9ae9908d89879e9981ca199e82cd1783
iv = dcd9c3d2daca55dca4af2aafa63aa3e9
```

### 7.5 padding

36 字节输入补成 48 字节：

```
native_input + 0x0c * 12
```

### 7.6 标准 AES-128-CBC

```
ciphertext = AES_CBC_Encrypt(key, iv, padded_native_input)
```

### 7.7 gate

qk9v 不是做 hash compare，也不是分段校验，而是直接把最终 48 字节密文和内置 target 比较。

target 为：

```
569670de6d7e270e7e27a189cec7082b
a1883f69796631adbd7c6d0fea9f281d
60f9d1277f1b007c36d631727753edcf
```

合并后：

```
569670de6d7e270e7e27a189cec7082ba1883f69796631adbd7c6d0fea9f281d60f9d1277f1b007
```

c36d631727753edcf

8. 如何逆出 flag

因为最终 gate 是“精确 ciphertext 比较”，所以逆向就很直接了：

1. 先把 target ciphertext 用真实 key / IV 做 AES-CBC 解密

2. 去掉 12 字节 0x0c padding

3. 得到 36 字节 native_input

4. 再把这 36 字节过一遍同一个 Rc4Warp

5. 因为它本质是 XOR 流变换，所以再次运行就能逆回原始 flag

### 8.1 解密 target ciphertext

解出来的 36 字节 payload 是：

```
2f3314c304c1fa86dbd85e331093d5959d7eae4bc2a903315194e53c9ca07babd8d8d743
```

### 8.2 再过一遍 Rc4Warp

最终得到：

```
53554354467b773331315f64306e335f7930755f6b6e30775f6833726d33355f6e30777d
```

按UTF-8解码

```
SUCTF{w311_d0n3_y0u_kn0w_h3rm35_n0w}
Exp:
```

```python
import argparse
import sys
import zipfile
from dataclasses import dataclass

from analyze_hermes_bundle import j1 as hermes_j1

try:
from Crypto.Cipher import AES as _RefAES
except Exception:
_RefAES = None

MASK32 = 0xFFFFFFFF

FNV_PRIME = 0x1000193
FNV_OFFSET = 0x811C9DC5
CRC_POLY = 0xEDB88320
HERMES_MAGIC = bytes.fromhex("c61fbc03")
HERMES_VERSION = 90

SBOX = bytes.fromhex(
"637c777bf26b6fc53001672bfed7ab76ca82c97dfa5947f0add4a2af9ca472c0"
"b7fd9326363ff7cc34a5e5f171d8311504c723c31896059a071280e2eb27b275"
"09832c1a1b6e5aa0523bd6b329e32f8453d100ed20fcb15b6acbbe394a4c58cf"
"d0efaafb434d338545f9027f503c9fa851a3408f929d38f5bcb6da2110fff3d2"
"cd0c13ec5f974417c4a77e3d645d197360814fdc222a908846eeb814de5e0bdb"
"e0323a0a4906245cc2d3ac629195e479e7c8376d8dd54ea96c56f4ea657aae08"
"ba78252e1ca6b4c6e8dd741f4bbd8b8a703eb5664803f60e613557b986c11d9ee"
"1f8981169d98e949b1e87e9ce5528df8ca1890dbfe6426841992d0fb054bb16"
)

RCON = (0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x1B, 0x36)

RC4WARP_KEY = b"TobeorNottobe"
KEY_TEXT = b"youknowwhatImean"
FLOW_TEXT = b"itsallintheflow!"
BUNDLE_SOURCE = "bundles/cache.snap.bundle"
TARGET_FLAG = "SUCTF{w311_d0n3_y0u_kn0w_h3rm35_n0w}"

TARGET_BLOCK0 = bytes.fromhex("569670de6d7e270e7e27a189cec7082b")

TARGET_BLOCK1 = bytes.fromhex("a1883f69796631adbd7c6d0fea9f281d")
TARGET_TAIL = bytes.fromhex("60f9d1277f1b007c36d631727753edcf")
TARGET_CIPHERTEXT = TARGET_BLOCK0 + TARGET_BLOCK1 + TARGET_TAIL

def u32(value: int) -> int:
return value & MASK32

def rol8(value: int, bits: int) -> int:
value &= 0xFF
bits &= 7
return ((value << bits) | (value >> (8 - bits))) & 0xFF

def xor_bytes(left: bytes, right: bytes) -> bytes:
if len(left) != len(right):
raise ValueError("xor operands must have equal length")
return bytes(a ^ b for a, b in zip(left, right))

def pkcs7_pad(data: bytes, block_size: int = 16) -> bytes:
pad = block_size - (len(data) % block_size)
if pad == 0:
pad = block_size
return data + bytes([pad]) * pad

def build_crc_table() -> list[int]:
table = []
for i in range(256):
x = i
for _ in range(8):
x = (x >> 1) ^ CRC_POLY if (x & 1) else (x >> 1)
table.append(u32(x))
return table

CRC_TABLE = build_crc_table()

def fnv1a32(data: bytes, seed: int = FNV_OFFSET) -> int:
h = seed
for b in data:
h = u32((h ^ b) * FNV_PRIME)
return h

def crc32_state(data: bytes, seed: int = MASK32) -> int:
h = seed
for b in data:
h = CRC_TABLE[(b ^ h) & 0xFF] ^ (h >> 8)
return u32(h)

def rc4warp_process(data: bytes, key: bytes) -> bytes:

if not key:
raise ValueError("key must not be empty")

s = list(range(256))
acc = 0
twist = 195
for i in range(256):
k1 = key[(5 * i + 1) % len(key)]
k2 = key[(3 * i + 7) % len(key)]
twist = rol8(twist, 1)
acc = (acc + s[i] + k1 + (k2 ^ twist) + i) & 0xFF
s[i], s[acc] = s[acc], s[i]

out = bytearray(len(data))
j = 0
k = 0
seed = 157
for idx, value in enumerate(data):
j = (j + 1) & 0xFF
a = s[j]
k = (k + a + 11 * j) & 0xFF
c = s[k]
s[j], s[k] = s[k], s[j]
mix = s[(j + k) & 0xFF]
t = (s[j] + a + ((mix ^ seed) & 0xFF)) & 0xFF
d = s[t]
seed = rol8(seed, 3)
e = s[(d ^ seed) & 0xFF]
out[idx] = value ^ d ^ e ^ ((13 * j) & 0xFF)
return bytes(out)

def rot_word(word: bytes) -> bytes:
return word[1:] + word[:1]

def sub_word(word: bytes) -> bytes:
return bytes(SBOX[b] for b in word)

def aes128_expand_key(key: bytes) -> tuple[bytes, ...]:
if len(key) != 16:
raise ValueError("AES-128 key must be 16 bytes")

words = [list(key[i:i + 4]) for i in range(0, 16, 4)]
for idx in range(4, 44):
temp = words[idx - 1][:]
if idx % 4 == 0:
temp = list(sub_word(rot_word(bytes(temp))))
temp[0] ^= RCON[idx // 4 - 1]

words.append([words[idx - 4][j] ^ temp[j] for j in range(4)])
return tuple(bytes(sum(words[4 * round_idx:4 * (round_idx + 1)], [])) for
round_idx in range(11))

def add_round_key(state: list[int], round_key: bytes) -> list[int]:
return [value ^ round_key[idx] for idx, value in enumerate(state)]

def sub_bytes_state(state: list[int]) -> list[int]:
return [SBOX[value] for value in state]

def shift_rows(state: list[int]) -> list[int]:
return [
state[0], state[5], state[10], state[15],
state[4], state[9], state[14], state[3],
state[8], state[13], state[2], state[7],
state[12], state[1], state[6], state[11],
]

def gf_mul(left: int, right: int) -> int:
result = 0
a = left & 0xFF
b = right & 0xFF
for _ in range(8):
if b & 1:
result ^= a
high = a & 0x80
a = (a << 1) & 0xFF
if high:
a ^= 0x1B
b >>= 1
return result

def mix_columns(state: list[int]) -> list[int]:
out = [0] * 16
for col in range(4):
idx = 4 * col
a0, a1, a2, a3 = state[idx:idx + 4]
out[idx + 0] = gf_mul(a0, 2) ^ gf_mul(a1, 3) ^ a2 ^ a3
out[idx + 1] = a0 ^ gf_mul(a1, 2) ^ gf_mul(a2, 3) ^ a3
out[idx + 2] = a0 ^ a1 ^ gf_mul(a2, 2) ^ gf_mul(a3, 3)
out[idx + 3] = gf_mul(a0, 3) ^ a1 ^ a2 ^ gf_mul(a3, 2)
return out

def aes128_encrypt_block(block: bytes, round_keys: tuple[bytes, ...]) -> bytes:
state = add_round_key(list(block), round_keys[0])
for round_idx in range(1, 10):
state = sub_bytes_state(state)

state = shift_rows(state)
state = mix_columns(state)
state = add_round_key(state, round_keys[round_idx])
state = sub_bytes_state(state)
state = shift_rows(state)
state = add_round_key(state, round_keys[10])
return bytes(state)

def aes_cbc_encrypt(key: bytes, iv: bytes, plaintext: bytes) -> bytes:
round_keys = aes128_expand_key(key)
prev = iv
blocks = []
for offset in range(0, len(plaintext), 16):
block = plaintext[offset:offset + 16]
enc = aes128_encrypt_block(xor_bytes(block, prev), round_keys)
blocks.append(enc)
prev = enc
return b"".join(blocks)

@dataclass
class HermesStage:
header_magic: str
bytecode_version: int
valid: bool
exported_entry: str
side_effect_only: bool

@dataclass
class BundleState:
fnv32: int
crc_state: int
crc_final: int
key: bytes
iv: bytes
round_keys: tuple[bytes, ...]
hermes: HermesStage

@dataclass
class EncryptionTrace:
user_input: bytes
rc4_output: bytes
padded_plaintext: bytes
ciphertext: bytes
target_ciphertext: bytes
target_match: bool

@dataclass

class RecoveryTrace:
ciphertext: bytes
decrypted_payload: bytes
recovered_flag: bytes

def model_hermes_stage(bundle: bytes) -> HermesStage:
version = int.from_bytes(bundle[8:12], "little") if len(bundle) >= 12 else
```

-1

```python
valid = len(bundle) >= 16 and bundle[:4] == HERMES_MAGIC and version ==
HERMES_VERSION
return HermesStage(
header_magic=bundle[:4].hex(),
bytecode_version=version,
valid=valid,
exported_entry="__j1",
side_effect_only=True,
)

def derive_key_iv(bundle: bytes) -> tuple[int, int, int, bytes, bytes]:
fnv32 = fnv1a32(bundle)
crc_state = crc32_state(bundle)
crc_final = u32(~crc_state)
size = len(bundle)

key = bytes(
bundle[(11 + 17 * idx) % size] ^ ((fnv32 + idx) & 0xFF) ^ KEY_TEXT[idx]
for idx in range(16)
)
iv = bytes(
bundle[(7 + 29 * idx) % size] ^ (((crc_final >> 8) + 3 * idx) & 0xFF)
^ FLOW_TEXT[idx]
for idx in range(16)
)
return fnv32, crc_state, crc_final, key, iv

def build_bundle_state(bundle: bytes) -> BundleState:
hermes = model_hermes_stage(bundle)
fnv32, crc_state, crc_final, key, iv = derive_key_iv(bundle)
return BundleState(
fnv32=fnv32,
crc_state=crc_state,
crc_final=crc_final,
key=key,
iv=iv,
round_keys=aes128_expand_key(key),
hermes=hermes,
)

def qk9v_encrypt_native_input(native_input: bytes, state: BundleState) ->
bytes:
if len(native_input) != 36:
raise ValueError("native input must be exactly 36 bytes")
return aes_cbc_encrypt(state.key, state.iv, pkcs7_pad(native_input, 16))

def qk9v_gate_exact(ciphertext: bytes) -> bool:
return ciphertext == TARGET_CIPHERTEXT

def recover_flag_from_target(state: BundleState) -> RecoveryTrace:
plain = _RefAES.new(state.key, _RefAES.MODE_CBC,
iv=state.iv).decrypt(TARGET_CIPHERTEXT)
if plain[-12:] != b"\x0c" * 12:
raise ValueError("target ciphertext does not decode to expected PKCS#7
padding")
payload = plain[:-12]
recovered = rc4warp_process(payload, RC4WARP_KEY)
return RecoveryTrace(
ciphertext=TARGET_CIPHERTEXT,
decrypted_payload=payload,
recovered_flag=recovered,
)

def encrypt_full_pipeline(user_input: bytes, state: BundleState) ->
EncryptionTrace:
if len(user_input) != 36:
raise ValueError("user input must be exactly 36 bytes")

rc4_output = rc4warp_process(user_input, RC4WARP_KEY)
padded_plaintext = pkcs7_pad(rc4_output, 16)
ciphertext = qk9v_encrypt_native_input(rc4_output, state)
return EncryptionTrace(
user_input=user_input,
rc4_output=rc4_output,
padded_plaintext=padded_plaintext,
ciphertext=ciphertext,
target_ciphertext=TARGET_CIPHERTEXT,
target_match=qk9v_gate_exact(ciphertext),
)

def load_bundle(apk_path: str | None, bundle_path: str | None) -> bytes:
if bundle_path:
with open(bundle_path, "rb") as handle:
return handle.read()
if apk_path is None:
raise ValueError("either apk_path or bundle_path is required")

with zipfile.ZipFile(apk_path) as zf:
return zf.read("assets/flutter_assets/bundles/cache.snap.bundle")

def dump_state(state: BundleState) -> None:
print("bundle fnv32 =", hex(state.fnv32))
print("bundle crc_state =", hex(state.crc_state))
print("bundle crc_final =", hex(state.crc_final))
print("actual key =", state.key.hex())
print("actual iv =", state.iv.hex())
print("round key[0] =", state.round_keys[0].hex())
print("round key[10] =", state.round_keys[-1].hex())
print("hermes valid =", state.hermes.valid)
print("hermes magic =", state.hermes.header_magic)
print("hermes version =", state.hermes.bytecode_version)
print("hermes entry =", state.hermes.exported_entry)
print("hermes eval only =", state.hermes.side_effect_only)

def dump_trace(trace: EncryptionTrace, include_hermes_preview: bool) -> None:
print("user input =", trace.user_input.hex(), trace.user_input)
print("rc4warp output =", trace.rc4_output.hex())
print("padded plaintext =", trace.padded_plaintext.hex())
print("ciphertext =", trace.ciphertext.hex())
print("target ciphertext =", trace.target_ciphertext.hex())
print("target match =", trace.target_match)
if include_hermes_preview:
print("hermes __j1(input) =", hermes_j1(trace.user_input[:16]))
for offset in range(0, len(trace.ciphertext), 16):
block_no = offset // 16
print(f"hermes __j1(cipher[{block_no}]) =
{hermes_j1(trace.ciphertext[offset:offset + 16])}")

def self_test(bundle: bytes, state: BundleState) -> None:
assert state.hermes.valid
assert len(state.key) == 16
assert len(state.iv) == 16
assert TARGET_CIPHERTEXT == TARGET_BLOCK0 + TARGET_BLOCK1 + TARGET_TAIL
assert qk9v_gate_exact(TARGET_CIPHERTEXT)

sample = bytes(range(36))
rc4 = rc4warp_process(sample, RC4WARP_KEY)
assert rc4warp_process(rc4, RC4WARP_KEY) == sample

padded = pkcs7_pad(rc4, 16)
cipher = qk9v_encrypt_native_input(rc4, state)
assert len(cipher) == 48
if _RefAES is not None:

ref = _RefAES.new(state.key, _RefAES.MODE_CBC,
iv=state.iv).encrypt(padded)
assert cipher == ref
recovered = recover_flag_from_target(state)
assert recovered.recovered_flag.decode("utf-8") == TARGET_FLAG

def main() -> int:
parser = argparse.ArgumentParser(description="Reconstruct the new
libjunk.so forward encryption pipeline.")
parser.add_argument("candidate", nargs="?", help="36-byte user input")
parser.add_argument("--apk", default="attachment/flumel.apk", help="APK
path used to load cache.snap.bundle")
parser.add_argument("--bundle", help="Override bundle path with a raw
cache.snap.bundle file")
parser.add_argument("--hermes", action="store_true", help="Print Hermes
__j1 previews for 16-byte blocks")
parser.add_argument("--recover-target", action="store_true", help="Decrypt
the fixed target ciphertext and recover the final flag")
parser.add_argument("--self-test", action="store_true", help="Run local
consistency checks")
args = parser.parse_args()

bundle = load_bundle(args.apk, args.bundle)
state = build_bundle_state(bundle)

if args.self_test:
self_test(bundle, state)

dump_state(state)

if args.recover_target:
if _RefAES is None:
print("PyCryptodome is required for target recovery mode",
file=sys.stderr)
return 1
recovered = recover_flag_from_target(state)
print("target payload =", recovered.decrypted_payload.hex())
print("recovered flag hex =", recovered.recovered_flag.hex())
print("recovered flag =", recovered.recovered_flag.decode("utf-
8"))

if not args.candidate:
return 0

data = args.candidate.encode()
if len(data) != 36:
print("input must be exactly 36 bytes", file=sys.stderr)

return 1

trace = encrypt_full_pipeline(data, state)
dump_trace(trace, args.hermes)
return 0

if __name__ == "__main__":
raise SystemExit(main())
```

Pwn

## 方法总结
- 核心技巧：Flutter/Dart/native/Hermes 混合链路逆向
- 识别信号：APK 中同时存在 Dart 逻辑、native so 和 Hermes bundle，输入跨层传递。
- 复用要点：先还原 Dart 输入变换，再分析 native 调用 bytecode 和 key/IV 派生，最后逆 AES-CBC 校验。
