# local_music

## 题目简述

题目给出 Rust 编译的 `wyy` 程序和自定义 `flag.enc` 音频容器。容器仿照 NCM 的分帧和流式异或思路：key frame 与 metadata frame 分别用 AES-128-ECB 保护，音频本体用 RC4-like 密钥流异或。两把 AES 密钥来自固定字符串和文件写入时的 Unix 时间戳，该时间戳又与容器的 mtime 对齐。

解密音频后还有一层频谱隐写：ID3/元数据中的 `Do you know FFT?` 和 `spectrum` 提示要查看频谱图，flag 文字被写在音频末段的频率布局中。本题虽然以隐写收尾，但前提是逆出二进制容器、密钥派生和非标准密钥流，所以决定性主障碍仍归类为 Reverse。

## 解题过程

### 还原容器格式

从 Rust 程序可还原出如下布局。所有长度都是 4 字节小端无符号整数：

```text
"MINILCTF\0\0"                         # 10-byte header
u32 key_len || key_frame
u32 meta_len || meta_frame
00 00 00 00 00
u32 image_offset
u32 image_len || image_data
encrypted_audio
```

key frame 的生成方式为：

```text
AES-128-ECB-PKCS7(core_key, b"miniL-audio-key" || audio_key)
然后每字节异或 0x64
```

metadata frame 为：

```text
b"miniL meta:" || Base64(
    AES-128-ECB-PKCS7(meta_key, b"miniL:" || metadata_json)
)
然后整体每字节异或 0x63
```

因此 `miniL-audio-key`、`miniL meta:` 和 `miniL:` 既是结构常量，也是枚举时的强校验信号。

### 时间戳密钥与音频密钥流

时间戳 `ts` 的密钥派生为：

```python
digest = hashlib.sha256(b"KaguyaIrohaYachiyo" + str(ts).encode()).digest()
core_key = digest[:16]
meta_key = digest[16:]
```

出题程序会反复写文件，直到实际 mtime 秒数等于用于加密的 `ts`。因此首先直接尝试 `int(flag.enc.stat().st_mtime)`；如果传输过程破坏了时间戳，再以可信基准为中心按 $0,+1,-1,+2,-2,\ldots$ 枚举。`build.rs` 还把编译时间桶范围编译进程序，但实际求解时 mtime 是更精确的窄化信息。

key frame 解出的 `audio_key` 是 64 字节。它在出题端由 16 字节 `core_key` 经 4 轮 rotate/add/xor 扩展得到，但解题时直接从 key frame 提取即可。音频的 KSA 与 RC4 相同，字节流索引则是题目定制的：

```python
def make_key_box(key: bytes) -> list[int]:
    box = list(range(256))
    last = 0
    for i in range(256):
        last = (box[i] + last + key[i % len(key)]) & 0xFF
        box[i], box[last] = box[last], box[i]
    return box


def decrypt_audio(ciphertext: bytes, key: bytes) -> bytes:
    box = make_key_box(key)
    plain = bytearray(ciphertext)
    for i, value in enumerate(plain):
        j = (i + 1) & 0xFF
        stream = box[(box[j] + box[(box[j] + j) & 0xFF]) & 0xFF]
        plain[i] = value ^ stream
    return bytes(plain)
```

枚举一个候选时间戳时，依次检查 PKCS#7 padding、`miniL-audio-key` 前缀、`miniL:` 前缀以及音频 magic（`fLaC`、`ID3`、`FF FB`、`OggS` 等）。这些条件能使错误密钥几乎立即被拒绝。

### 从频谱中读取 flag

解密后得到 MP3。用 Audacity 或 Sonic Visualiser 切换为 Spectrogram 视图，查看音频末段，即可看到由频率空洞组成的文字。若用脚本复现，可先用 ffmpeg 转为 48 kHz WAV，然后以 `nperseg=4096`、`noverlap=3840` 做 STFT，把幅度转为 dB 并翻转频率轴。出题脚本默认把文字放在结尾前约 18–38 秒、1.2–13 kHz 的范围。

频谱中的结果为：

```text
miniL{Yur1_1s_JusT1c3!!F0ll0w_Ch0u-K@guy@-h1me!th@nks_m03w}
```

## 方法总结

- 核心技巧：通过逆向 Rust 程序恢复自定义分帧容器，利用文件 mtime 重建 AES 密钥，解出 RC4-like 音频后再检查频谱隐写。
- 识别信号：附件同时出现未知容器、时间戳派生密钥、可校验的明文前缀，并在音频元数据中出现 FFT/spectrum 提示时，应串联“容器逆向 → 密钥枚举 → 音频频谱”两阶段。
- 复用要点：不要盲目遍历编译时间桶的巨大范围，先用 mtime 窄化搜索；枚举时应同时使用 padding、帧前缀和文件 magic 做强验证。
