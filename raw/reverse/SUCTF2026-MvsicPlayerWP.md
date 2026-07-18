# SUCTF2026-MvsicPlayer

## 题目简述

题目给出一个 Electron 音乐播放器和被“安全归档”后的 `ddd.su_mv_enc`。播放器只接受自定义 `.su_mv` 容器：前端先解析容器、得到可播放的 WAV payload；开启安全模式并播放结束或关闭窗口后，主进程再调用原生 Node.js addon 加密这个 payload。

题目最终的提交格式是 `SUCTF{md5(wav 文件)}`，因此不必还原存在多种合法编码的 `.su_mv` 容器，只需逆向 `vm_encryptor.node` 的 WAV 专用分支，恢复原始 WAV 并计算 MD5。

## 解题过程

### 1. 理清 Electron 数据流

先从 `win-unpacked/resources/app.asar` 提取应用代码：

```powershell
npx asar extract "win-unpacked/resources/app.asar" "app_asar_extracted"
```

官方题解提到可以用在线 JS 反混淆器处理打包后的脚本。这里的混淆只包裹了字符串和标识符，任何能完成格式化、常量传播和字符串解码的本地或在线工具都可替代；具体网站并非解题依据，因此不保留易失的工具链接。

关键调用链如下：

```text
renderer/app.js
  -> SUMV.parseSuMv(fileBytes)
  -> parsed.payload
  -> updateSession({ currentPayload: parsed.payload })
  -> main.js: vmEncrypt(currentPayload)
  -> 原文件被删除，密文写入 <原路径>_enc
```

`src/common/sumv-browser.js` 同时公开了 `.su_mv` 的容器格式：

```text
0x00  4 bytes  "SUMV"
0x04  1 byte   version
0x06  1 byte   formatCode
0x08  u32le    解码后的 payload 长度
0x0c  u32le    压缩数据长度
0x10  ...      自定义压缩数据
```

压缩数据先经过四类 token（literal、repeat、等差序列和 1 KiB 窗口回溯）解码，再用密钥 `SUMUSICPLAYER` 做 RC4 变换，结果才是 WAV payload。`main.js` 保存到会话并交给 addon 的正是这个 payload，所以 `ddd.su_mv_enc` 不是整个 `.su_mv` 文件的密文。

### 2. 排除 JS 回退算法

`src/main/native-bridge.js` 中能看到一个很简单的 `placeholderVmEncrypt()`：维护单字节状态，对每个输入字节异或后循环左移，并在结果前加 `SVE4`。但只要 `vm_encryptor.node` 存在，桥接层就优先调用 addon；placeholder 只是模块不可用时的回退实现。

从 addon 导出的 `napi_register_module_v1` 追到注册名 `vmEncrypt`，可定位真正回调。它先严格解析内存中的 WAV，要求：

- 文件头为 `RIFF`，格式为 `WAVE`；
- 同时存在合法的 `fmt ` 与 `data` chunk；
- `audioFormat == 1`，即未压缩 PCM；
- `bitsPerSample == 16`；
- 声道数为 1～8，且 `blockAlign == channels * 2`；
- `data` 长度非零并能被 `blockAlign` 整除。

不满足条件时才走逐字节 placeholder；合法 WAV 则进入 VM 实现的专用 SVE4 分支。参赛队题解用黑盒实验作了同样的区分：随机数据的原生输出与 placeholder 一致，而合法 WAV 的输出不同，并且去掉 `SVE4` 后总是按 64 字节对齐。

### 3. 还原 VM 中的分组算法

WAV 分支会生成、编译并执行一段固定 VM bytecode。VM 是普通栈机，支持立即数、寄存器、算术/位运算、跳转以及 8/16/32/64 位内存读写；真正重要的是 bytecode 所实现的高层流程：

1. 对完整 WAV 做 64 字节 PKCS#7 padding；
2. 每块按大端序拆成 16 个 32 位字，分成各 8 字的 `L`、`R`；
3. 使用 32 字节初始密钥 `00 01 ... 1f`，执行 4 轮自定义 Feistel/Speck 混合；
4. 写出 64 字节密文；
5. 令下一块密钥为本块密文前、后 32 字节逐字节异或的结果；
6. 最终在全部密文前加 ASCII 头 `SVE4`。

块间只有密钥链依赖，而且下一块密钥完全由已知的当前密文导出，因此可以从第一块开始顺序解密，无需猜测 IV。每轮中先更新 8 个密钥字并派生 12 个轮密钥，再对右半部的四对字做 Speck 风格变换；Feistel 结构使其可以按轮倒序恢复。

下面的脚本保留官方 `exp.py` 的核心解密逻辑，并补上头部、padding、WAV 魔数和最终哈希校验：

```python
import hashlib
import struct
from pathlib import Path

MASK32 = 0xFFFFFFFF
BLOCK_SIZE = 64
DELTA = 0x70336364


def rol32(x, r):
    return ((x << r) | (x >> (32 - r))) & MASK32


def ror32(x, r):
    return ((x >> r) | (x << (32 - r))) & MASK32


def words_be(data):
    return list(struct.unpack(">" + "I" * (len(data) // 4), data))


def bytes_be(words):
    return struct.pack(">" + "I" * len(words), *words)


def expand_key(key, rounds=4):
    a, b, c, d, e, f, g, h = words_be(key)
    ksum = 0x73756572
    result = []

    for rnd in range(rounds):
        ksum = (ksum + DELTA + rnd) & MASK32
        a = (a + rol32(b ^ ksum, 3)) & MASK32
        b = (b + rol32(c ^ a, 5)) & MASK32
        c = (c + rol32(d ^ b, 7)) & MASK32
        d = (d + rol32(e ^ c, 11)) & MASK32
        e = (e + rol32(f ^ d, 13)) & MASK32
        f = (f + rol32(g ^ e, 17)) & MASK32
        g = (g + rol32(h ^ f, 19)) & MASK32
        h = (h + rol32(a ^ g, 23)) & MASK32

        result.append([
            a ^ c ^ ksum,
            b ^ d ^ ((ksum + 0x62616F7A) & MASK32),
            e ^ g ^ ((ksum + 0x6F6E6777) & MASK32),
            f ^ h ^ ((ksum + 0x696E6221) & MASK32),
            (a + e) & MASK32,
            (b + f) & MASK32,
            (c + g) & MASK32,
            (d + h) & MASK32,
            a ^ f,
            b ^ g,
            c ^ h,
            d ^ e,
        ])
    return result


def inverse_speck_pairs(mixed, round_key):
    original = mixed[:]
    for pair in range(4):
        i = pair * 2
        x1, y1 = mixed[i], mixed[i + 1]
        y0 = ror32(y1 ^ x1, 3)
        x0 = rol32(((x1 ^ round_key[pair]) - y0) & MASK32, 8)
        original[i], original[i + 1] = x0, y0
    return original


def round_f(mixed, round_key, round_sum):
    output = []
    for i in range(8):
        value = (
            ((((mixed[i] << 4) & MASK32) ^ (mixed[i] >> 5))
             + mixed[(i + 1) & 7])
            & MASK32
        )
        value ^= (round_sum + round_key[4 + i]) & MASK32
        extra = rol32(mixed[(i + 3) & 7], i + 1)
        value = (value + (extra ^ (round_sum >> ((i + 1) & 7)))) & MASK32
        output.append(value)
    return output


def decrypt_block(block, key):
    words = words_be(block)
    left, right = words[:8], words[8:]
    round_keys = expand_key(key)
    round_sums = [DELTA * (i + 1) & MASK32 for i in range(4)]

    for rnd in range(3, -1, -1):
        mixed = left
        f_value = round_f(mixed, round_keys[rnd], round_sums[rnd])
        previous_left = [right[i] ^ f_value[i] for i in range(8)]
        previous_right = inverse_speck_pairs(mixed, round_keys[rnd])
        left, right = previous_left, previous_right

    return bytes_be(left + right)


def decrypt_file(path):
    blob = Path(path).read_bytes()
    if blob[:4] != b"SVE4":
        raise ValueError("错误的 SVE4 文件头")

    ciphertext = blob[4:]
    if not ciphertext or len(ciphertext) % BLOCK_SIZE:
        raise ValueError("密文长度未按 64 字节对齐")

    key = bytes(range(32))
    plaintext = bytearray()
    for offset in range(0, len(ciphertext), BLOCK_SIZE):
        block = ciphertext[offset:offset + BLOCK_SIZE]
        plaintext.extend(decrypt_block(block, key))
        key = bytes(a ^ b for a, b in zip(block[:32], block[32:]))

    pad = plaintext[-1]
    if not 1 <= pad <= BLOCK_SIZE or plaintext[-pad:] != bytes([pad]) * pad:
        raise ValueError("PKCS#7 padding 校验失败")
    wav = bytes(plaintext[:-pad])
    if wav[:4] != b"RIFF" or wav[8:12] != b"WAVE":
        raise ValueError("解密结果不是 WAV")
    return wav


wav = decrypt_file("ddd.su_mv_enc")
Path("ddd.wav").write_bytes(wav)
digest = hashlib.md5(wav).hexdigest()
print(digest)
print(f"SUCTF{{{digest}}}")
```

运行后恢复出 `ddd.wav`，其 MD5 为 `16ac79d3510d6ea4b5338fade80459b8`，因此提交：

```text
SUCTF{16ac79d3510d6ea4b5338fade80459b8}
```

### 4. 为什么不再重建 `.su_mv`

早期题面要求提交原始 `.su_mv` 的 MD5，但容器压缩层允许多种 token 组合表示同一 payload，无法由 WAV 唯一反推出原始容器字节。官方题解也明确说明这里会产生多解，最终才把目标改为 WAV 的 MD5。因此，重新压缩 WAV 并生成一个“等价 `.su_mv`”只能证明播放器可接受该文件，不能证明它与原始文件逐字节一致；在当前题面下这一步没有必要。

## 方法总结

- Electron 题先从 `app.asar` 还原 IPC 与数据流，明确 native addon 接收的是容器、解码后 payload，还是其它中间态。
- 不要把桥接层的 fallback 当成真实算法；应通过模块加载条件、输入格式分支和输出长度共同判断实际路径。
- 面对大段 VM bytecode，先恢复 padding、分组宽度、字节序、轮函数和块间状态等高层结构，再实现逆运算，比逐条复刻全部 9199 条解释执行轨迹更直接。
- 对链式分组模式，先确认下一块状态依赖明文还是密文。本题的下一块密钥只依赖当前密文，因此可以顺序解密。
- 最终提交目标是 WAV 的 MD5；等价容器不等于原始容器，不能把“能播放”误当成“字节级还原”。
