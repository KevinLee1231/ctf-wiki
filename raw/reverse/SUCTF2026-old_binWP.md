# SUCTF2026-old_bin

## 题目简述

题目给出老式固件 `old.bin`。文件先被单字节 XOR，再封装为自定义 `IMG0` 容器；主段解压后是被破坏的 MIPS64 ELF。修复 ELF 后可以看到一个监听 TCP 5534 端口的校验服务：服务端发送 32 字节 challenge，接收最多 64 字节响应，再经过确定性 PRNG、两层字节变换和自定义分组密码校验 flag。

官方仓库的 `SU_老年固件` 目录只包含附件和 `flag.txt`，没有单题 WP 或源码。因此以下过程以附件静态分析为准，并逐页对照总 PDF 中第 116–134 页的参赛队题解复核。

## 解题过程

### 1. 去除外层 XOR 并拆出 IMG0 段

`old.bin` 中 `0x7f` 异常密集。全文件异或 `0x7f` 后，开头恢复为 `IMG0`，头部同时给出了三个段的偏移和长度。三个段均为 XZ 数据，主逻辑位于第一个大段：

```python
import lzma
from pathlib import Path

blob = bytes(b ^ 0x7f for b in Path("old.bin").read_bytes())
Path("old_xor.bin").write_bytes(blob)

segments = [
    (0x2028, 0x4EEAC, "seg1"),
    (0x50ED4, 0x0BD0, "seg2"),
    (0x51AA4, 0x1408, "seg3"),
]

for offset, size, name in segments:
    packed = blob[offset:offset + size]
    Path(f"{name}.bin").write_bytes(packed)
    Path(f"{name}.out").write_bytes(lzma.decompress(packed))
```

### 2. 修复 ELF 头和 program header

`seg1.out` 实际是 64 位小端 MIPS ELF，但前四字节不是标准 magic。恢复 `7f 45 4c 46` 后，ELF 已可识别，不过第二个 `PT_LOAD` 的 `p_offset` 被故意减去了 `0x10000`，依赖它的 TLS program header 也发生同样错位。对应 program header 索引为 2 和 5：

```python
import struct
from pathlib import Path

elf = bytearray(Path("seg1.out").read_bytes())
elf[:4] = b"\x7fELF"

phoff = struct.unpack_from("<Q", elf, 0x20)[0]
phentsize = struct.unpack_from("<H", elf, 0x36)[0]
for index in (2, 5):
    entry = phoff + index * phentsize
    old_offset = struct.unpack_from("<Q", elf, entry + 8)[0]
    struct.pack_into("<Q", elf, entry + 8, old_offset + 0x10000)

Path("seg1_fixed_all.elf").write_bytes(elf)
```

修复后应再检查 ELF header、各 `PT_LOAD` 的文件区间以及 TLS 是否落入对应加载段；只修 magic 而不修偏移，会让反编译器错误映射全局表和 TLS 状态。

### 3. 还原网络校验的三层结构

主函数创建 socket、绑定 `0.0.0.0:5534` 并监听。每个连接的处理流程可以整理为：

```text
确定性 PRNG 生成 32 字节 challenge -> send
recv 最多 64 字节响应
响应补齐/混淆为 64 字节
逐字节 6 轮非线性变换
置换 + AES S-box 层
四个 16 字节自定义分组密码块
与 ELF 中的 64 字节 target 比较
```

初始化函数从四个 64 位种子出发，经 SplitMix64 生成 xoshiro256** 状态，再依次生成：

- `tbl20`：64 字节掩码表；
- `tbl28`：Fisher–Yates 得到的 64 项置换；
- `tbl30`：48 字节轮掩码。

第一层先把输入补为 64 字节。已有输入使用原字节，超出接收长度的位置使用 `17 * i`，随后异或 `tbl20[(7*i) & 0x3f] + i`。接着每个位置独立执行 6 轮：

```text
x ^= (prng_value + position + round) & 0xff
x  = rol8(x, 1)
x ^= AES_SBOX[(x + 13*round) & 0xff]
```

该映射不保证一一对应，逆向时不能假设存在唯一逆函数；最稳妥的方法是对每个位置枚举全部 256 个输入字节，建立输出到输入候选的反向表。

第二层按 `tbl28` 重排，再依次异或 `tbl30`、查标准 AES S-box、异或 `tbl20`。这一层用 AES 逆 S-box 即可直接还原。

最后一层每 16 字节一块，结构类似被修改的 SM4：

- 主密钥、FK、32 项 CK、自定义 S-box 和 64 字节 target 均位于修复后的 ELF；
- S-box 实际索引为 `(byte + 0x37) & 0xff`；
- 共执行 34 轮，轮密钥通过 `round & 31` 复用前两项，不能误写成普通 SM4 的 32 轮；
- 第 8、16、24 轮后还有额外异或白化；
- 输入、输出存在固定 `0xAAAAAAAA` 及四个 32 位常量的仿射处理。

由于轮结构是四字状态左移并生成新字，按轮倒序即可从 target 恢复第二层输出。

### 4. 离线恢复 flag

下面给出整理后的核心求解器。它直接从修复 ELF 读取常量，重建初始化上下文，依次逆三层变换；对第一层产生的多候选使用 flag 前后缀和字符集过滤，最后重新正向计算 target，避免仅凭“可打印字符”选错分支。

```python
from dataclasses import dataclass
from itertools import product
from pathlib import Path

MASK64 = (1 << 64) - 1
MASK32 = (1 << 32) - 1

AES_SBOX_OFF = 0x7E6C0
TARGET_OFF = 0x7E7C0
KEY_OFF = 0x7E920
FK_OFF = 0x7E950
CK_OFF = 0x7E970
CUSTOM_SBOX_OFF = 0x7EA70

SEEDS = [
    0xFFF55731369D7563,
    0x16E58EB22FBD5C72,
    0x3632ED844C43F5B0,
    0x390980A442221584,
]


@dataclass
class Constants:
    aes: list[int]
    aes_inv: list[int]
    target: bytes
    key: bytes
    fk: list[int]
    ck: list[int]
    sbox: list[int]


@dataclass
class Context:
    state: list[int]
    mask: list[int]
    permutation: list[int]
    round_mask: list[int]


def rol64(x, n):
    return ((x << n) & MASK64) | (x >> (64 - n))


def rol32(x, n):
    return ((x << n) & MASK32) | (x >> (32 - n))


def rol8(x, n):
    return ((x << n) & 0xFF) | (x >> (8 - n))


def splitmix64(box):
    box[0] = (box[0] + 0x9E3779B97F4A7C15) & MASK64
    z = box[0]
    z = ((z ^ (z >> 30)) * 0xBF58476D1CE4E5B9) & MASK64
    z = ((z ^ (z >> 27)) * 0x94D049BB133111EB) & MASK64
    return (z ^ (z >> 31)) & MASK64


def prng_next(state):
    s0, s1, s2, s3 = state
    result = (rol64((s1 * 5) & MASK64, 7) * 9) & MASK64
    temp = (s1 << 17) & MASK64
    s2 ^= s0
    s3 ^= s1
    s1 ^= s2
    s0 ^= s3
    s2 ^= temp
    s3 = rol64(s3, 45)
    return result, [s0 & MASK64, s1 & MASK64, s2 & MASK64, s3 & MASK64]


def load_constants(path):
    data = Path(path).read_bytes()
    aes = list(data[AES_SBOX_OFF:AES_SBOX_OFF + 256])
    aes_inv = [0] * 256
    for i, value in enumerate(aes):
        aes_inv[value] = i
    read64 = lambda off, count: [
        int.from_bytes(data[off + 8*i:off + 8*i + 8], "little")
        for i in range(count)
    ]
    return Constants(
        aes,
        aes_inv,
        data[TARGET_OFF:TARGET_OFF + 64],
        data[KEY_OFF:KEY_OFF + 16],
        read64(FK_OFF, 4),
        read64(CK_OFF, 32),
        list(data[CUSTOM_SBOX_OFF:CUSTOM_SBOX_OFF + 256]),
    )


def init_context(constants):
    mixer = [0x1234567890ABCDEF]
    state = []
    for seed in SEEDS:
        mixer[0] ^= (seed + 0x9E3779B97F4A7C15) & MASK64
        state.append(splitmix64(mixer))
    if not any(state):
        state[0] = 0xDEADBEEFCAFEBABE

    mask = [0] * 64
    permutation = list(range(64))
    round_mask = [0] * 48
    for i in range(64):
        value, state = prng_next(state)
        mask[i] = ((value & 0xFF) ^ ((value >> 11) & 0xFF)
                   ^ ((i - 0x5B) & 0xFF)) & 0xFF
    for i in range(63, 0, -1):
        value, state = prng_next(state)
        j = value % (i + 1)
        permutation[i], permutation[j] = permutation[j], permutation[i]
    for i in range(48):
        value, state = prng_next(state)
        temp = ((value & 0xFF) ^ ((value >> 23) & 0xFF)
                ^ (((7*i) & 0xFF) + 0x3D)) & 0xFF
        temp = constants.aes[(temp + mask[i & 0x3F]) & 0xFF]
        value2, state = prng_next(state)
        temp ^= value2 & 0xFF
        round_mask[i] = rol64(temp, i % 7 + 1) & 0xFF
    return Context(state, mask, permutation, round_mask)


def tau(word, constants):
    output = 0
    for shift in (24, 16, 8, 0):
        value = constants.sbox[(((word >> shift) & 0xFF) + 0x37) & 0xFF]
        output |= value << shift
    return output


def t_key(word, constants):
    value = tau(word, constants)
    return (value ^ rol32(value, 15) ^ rol32(value, 23)
            ^ 0xCAFEBABE) & MASK32


def t_round(word, constants):
    value = tau(word, constants)
    return (value ^ rol32(value, 3) ^ rol32(value, 11)
            ^ rol32(value, 19) ^ rol32(value, 27)
            ^ 0x12345678) & MASK32


def key_schedule(constants):
    mk = [int.from_bytes(constants.key[i:i + 4], "big") for i in range(0, 16, 4)]
    base = [((mk[i] ^ constants.fk[i]) + i) & MASK32 for i in range(4)]
    keys = []
    for i in range(32):
        previous = base[i] if i < 4 else keys[i - 4]
        tail = (base[i + 1:] + keys)[:3] if i < 4 else keys[i - 3:i]
        # 前四轮分别使用剩余 base 字和已生成轮密钥。
        if i == 0:
            mix = base[1] ^ base[2] ^ base[3]
        elif i == 1:
            mix = base[2] ^ base[3] ^ keys[0]
        elif i == 2:
            mix = base[3] ^ keys[0] ^ keys[1]
        elif i == 3:
            mix = keys[0] ^ keys[1] ^ keys[2]
        else:
            mix = keys[i - 3] ^ keys[i - 2] ^ keys[i - 1]
        value = previous ^ t_key(mix ^ constants.ck[i], constants)
        keys.append(((value + i) if i >= 4 else value) & MASK32)
    return keys


def bytes_to_words(block):
    return [int.from_bytes(block[i:i + 4], "big") for i in range(0, 16, 4)]


def words_to_bytes(words):
    return b"".join((word & MASK32).to_bytes(4, "big") for word in words)


def round_function(a, b, c, d, key, constants):
    return ((a ^ t_round(b ^ c ^ d ^ key, constants)) + 0x1337) & MASK32


def decrypt_block(block, constants, keys):
    y0, y1, y2, y3 = bytes_to_words(block)
    state = [
        y3 ^ 0x87654321,
        y2 ^ 0x10FEDCBA,
        y1 ^ 0xABCDEF01,
        y0 ^ 0x12345678,
    ]
    for rnd in range(33, -1, -1):
        if rnd in (8, 16, 24):
            state[0] ^= 0x55555555
            state[1] ^= 0xAAAAAAAA
        b, c, d, new = state
        old = (((new - 0x1337) & MASK32)
               ^ t_round(b ^ c ^ d ^ keys[rnd & 31], constants))
        state = [old & MASK32, b, c, d]
    return words_to_bytes([word ^ 0xAAAAAAAA for word in state])


def decrypt_target(constants):
    keys = key_schedule(constants)
    return b"".join(
        decrypt_block(constants.target[i:i + 16], constants, keys)
        for i in range(0, 64, 16)
    )


def inverse_second_layer(data, context, constants):
    output = [0] * 64
    for i in range(64):
        value = constants.aes_inv[data[i] ^ context.mask[i]]
        value ^= context.round_mask[i % 48]
        output[context.permutation[i] & 0x3F] = value
    return output


def six_round_values(context):
    state = context.state[:]
    values = []
    for _ in range(6):
        value, state = prng_next(state)
        values.append(value & 0x3F)
    return values


def first_transform(value, position, values, aes):
    for rnd, random_value in enumerate(values):
        value ^= (random_value + position + rnd) & 0xFF
        value = rol8(value, 1)
        value ^= aes[(value + 13*rnd) & 0xFF]
    return value & 0xFF


def inverse_first_layer(data, context, constants):
    values = six_round_values(context)
    candidates = []
    for position in range(64):
        wanted = data[position]
        mask = (context.mask[(7*position) & 0x3F] + position) & 0xFF
        candidates.append(sorted({
            value ^ mask
            for value in range(256)
            if first_transform(value, position, values, constants.aes) == wanted
        }))
    return candidates


def encrypt_block(block, constants, keys):
    state = [word ^ 0xAAAAAAAA for word in bytes_to_words(block)]
    for rnd in range(34):
        new = round_function(*state, keys[rnd & 31], constants)
        state = [state[1], state[2], state[3], new]
        if rnd in (8, 16, 24):
            state[0] ^= 0x55555555
            state[1] ^= 0xAAAAAAAA
    final = [
        state[3] ^ 0x12345678,
        state[2] ^ 0xABCDEF01,
        state[1] ^ 0x10FEDCBA,
        state[0] ^ 0x87654321,
    ]
    return words_to_bytes(final)


def forward_validate(flag, context, constants):
    values = six_round_values(context)
    first = []
    for i, byte in enumerate(flag):
        mask = (context.mask[(7*i) & 0x3F] + i) & 0xFF
        first.append(first_transform(byte ^ mask, i, values, constants.aes))

    second = [0] * 64
    for i in range(64):
        value = first[context.permutation[i] & 0x3F]
        value = constants.aes[value ^ context.round_mask[i % 48]]
        second[i] = value ^ context.mask[i]

    keys = key_schedule(constants)
    result = b"".join(
        encrypt_block(bytes(second[i:i + 16]), constants, keys)
        for i in range(0, 64, 16)
    )
    return result == constants.target


constants = load_constants("seg1_fixed_all.elf")
context = init_context(constants)
second = inverse_second_layer(decrypt_target(constants), context, constants)
choices = inverse_first_layer(second, context, constants)

allowed = set(map(ord, "abcdefghijklmnopqrstuvwxyz0123456789{}_"))
prefix, suffix = b"flag{", b"}"
for i in range(64):
    choices[i] = set(choices[i]) & allowed
    if i < len(prefix):
        choices[i] &= {prefix[i]}
    if i >= 64 - len(suffix):
        choices[i] &= {suffix[i - (64 - len(suffix))]}

count = 1
for values in choices:
    count *= len(values)
if not count or count > 100000:
    raise RuntimeError(f"候选组合数异常：{count}")

for candidate in product(*[sorted(values) for values in choices]):
    flag = bytes(candidate)
    if forward_validate(flag, context, constants):
        print(flag.decode())
```

脚本得到唯一的 64 字节响应：

```text
flag{3putis6omqi3u7034722576kpze4udduejoko8zr3e6ozvp8mosm6065q1}
```

该结果同时与官方仓库内的 `flag.txt` 一致。

## 方法总结

- 高频单字节常量常提示整文件 XOR；恢复魔数后还要继续检查容器表和内嵌压缩格式。
- ELF 能被识别不代表映射正确。program header、TLS 和全局常量区错位时，应先修结构再相信反编译结果。
- 网络服务中的随机量若由硬编码种子和确定性 PRNG 生成，可以完全离线重建，无需运行目标服务。
- 对非双射的逐字节变换，应枚举 256 个输入建立逆映射，再用格式约束和完整正向校验消歧。
- 自定义“类 SM4”必须以实际轮数、轮密钥索引和额外白化为准；套用标准 32 轮模板会得到接近但错误的结果。
