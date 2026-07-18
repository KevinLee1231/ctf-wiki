# SUCTF2026-chaos

## 题目简述

附件 `attachment.zip` 使用传统 ZipCrypto 加密，包含 `challenge.avif` 和无扩展名的 `task`。题目的材料链较长：先利用 AVIF 固定文件头做 ZipCrypto 已知明文攻击；再分别从 AVIF 动画、WAV 双声道、DeepSound 隐写、WAV 尾部 ZIP 和反切密码中取出后续密钥；最后解析一条 WinZip AES 的 `$zip2$` hash，直接解开其中内嵌的小文件数据。

决定第一步能否成功的关键不是盲猜 ZIP 密码，而是避开 AVIF 文件头中不稳定的 box size，从偏移 4 开始构造足够长的连续已知明文。

## 解题过程

### 1. 用 AVIF 文件头攻击 ZipCrypto

AVIF 基于 ISOBMFF。文件开头是 `ftyp` box：

```text
offset 0x00: 4-byte box size        # 随文件而变
offset 0x04: "ftyp"
offset 0x08: major brand
offset 0x0c: minor version
offset 0x10: compatible brands...
```

静态 AVIF 常见 major brand 为 `avif`，动画序列则为 `avis`。本题从偏移 4 开始的实际字节为：

```text
66 74 79 70 61 76 69 73 00 00 00 00 61 76 69 73
 f  t  y  p  a  v  i  s              a  v  i  s
```

因此可直接交给 `bkcrack`：

```powershell
bkcrack -C attachment.zip -c challenge.avif -x 4 66747970617669730000000061766973
```

恢复出的三个 ZipCrypto 内部 key 为：

```text
b76b3323 6eebbce4 00a94706
```

用这组 key 解开外层 ZIP（也可用 `bkcrack -U` 重打包成一个自定密码的 ZIP），得到 `challenge.avif` 和 `task`。`task` 的开头是 `RIFF....WAVE`，应按 WAV 继续分析；其文件尾还能找到 `PK\x03\x04`，可另行 carve 出 `flag.zip`。归档注释明确提示其密码是 `MD5(secret.txt's answer)`。

![WAV 尾部的 flag.zip 与密码提示](SUCTF2026-chaosWP/suchaos.png)

### 2. AVIF 动画给出最终 AES key

`challenge.avif` 同时含主图和一个 5 帧、约 5 秒的动画流。先确认流信息，再导出第二路视频：

```bash
ffprobe -v error -show_streams -select_streams v challenge.avif
ffmpeg -i challenge.avif -map 0:v:1 stream1_%02d.png
montage stream1_02.png stream1_03.png stream1_04.png stream1_05.png \
  -tile 2x2 -geometry +0+0 hanxin.png
```

四个有效分片拼成的是汉信码，而不是普通 QR Code。用支持 Han Xin Code 的条码解码器读取，得到 64 个十六进制字符，即一把 256 位 AES key：

```text
0f87b6f831b312a0b6748c4a792b9362c033c75cc230aae63be2c9cfab12a0e4
```

此时还不知道它对应哪段数据，先保存，继续分析 WAV。

### 3. WAV、DeepSound 与反切密码

`task` 的左右声道大部分内容反相。把双声道相加为单声道后，背景会相互抵消，隐藏的 500 Hz / 900 Hz 摩尔斯节奏会变得明显。Audacity 中可拆分立体声后混音渲染，也可直接相加采样：

```python
import struct, wave

with wave.open('task', 'rb') as src, wave.open('mono.wav', 'wb') as dst:
    dst.setparams((1, src.getsampwidth(), src.getframerate(), 0, 'NONE', ''))
    for _ in range(src.getnframes()):
        left, right = struct.unpack('<hh', src.readframes(1))
        sample = max(-32768, min(32767, left + right))
        dst.writeframesraw(struct.pack('<h', sample))
```

结合频谱图按摩尔斯码读取，明文是：

```text
SUPERIDOL
```

以它作为 DeepSound 密码，可从原始 WAV 中提取 `secret.txt`。文件给出两首各 28 字的诗和七组三元组，并提示“以诗做表相切，一二三四，阴阳上去，定为声调”。完整规则是：

- 第一首诗按字序建立声母表；
- 第二首诗按字序建立韵母表；
- 三元组 `(a, b, t)` 取第 `a` 个字的声母、第 `b` 个字的韵母，再按 `t` 标声调；
- `a = 0` 表示零声母。

```text
3-21-1   -> yī
10-21-4  -> rì
13-7-4   -> kàn
2-9-4    -> jìn
15-15-2  -> cháng
0-28-1   -> ān
28-22-1  -> huā
```

所以答案为 `一日看尽长安花`。对这七个汉字的 UTF-8 字节计算 MD5：

```bash
printf '一日看尽长安花' | md5sum
2e4dc1dad6e4c0747371a041cb177dd7  -
```

该 MD5 就是从 WAV 尾部 carve 出的 `flag.zip` 的解压密码。解压后得到 `flag.txt`，内容不是 flag，而是一整条 WinZip AES hash：

```text
$zip2$*0*3*0*ee1f6cc09449ea4174cb45bd0d667d1c*258b*1c*0a6bd41815d0d2af8b30c25ce506b2ead194b0f3c4186913c80d2a2b*408973cbd18faafa7355*$/zip2$
```

### 4. 从 `$zip2$` hash 中直接恢复数据

目标文件只有几十字节，因此 zip2john 生成的 hash 已把整段压缩后密文内嵌在字符串中，这就是本题的 “data in hash”。字段中的有效密文为：

```text
0a6bd41815d0d2af8b30c25ce506b2ead194b0f3c4186913c80d2a2b
```

AVIF 汉信码给出的不是待爆破密码，而是这段 WinZip AES 数据实际使用的 32 字节加密 key。WinZip AES 使用小端计数器、初值 1 的 AES-CTR；解密后再按 raw DEFLATE（`wbits=-15`）解压：

```python
import zlib
from Crypto.Cipher import AES
from Crypto.Util import Counter

key = bytes.fromhex(
    '0f87b6f831b312a0b6748c4a792b9362'
    'c033c75cc230aae63be2c9cfab12a0e4'
)
ct = bytes.fromhex(
    '0a6bd41815d0d2af8b30c25ce'
    '506b2ead194b0f3c4186913c80d2a2b'
)
ctr = Counter.new(128, initial_value=1, little_endian=True)
compressed = AES.new(key, AES.MODE_CTR, counter=ctr).decrypt(ct)
print(zlib.decompress(compressed, -15).decode())
```

得到：

```text
SUCTF{f4ll1g_t0_the_C6a0s}
```

## 方法总结

本题串联了五种载体，但每一步都有明确的结构信号：AVIF 的 `ftyp/avis` 提供 ZipCrypto 连续已知明文；动画帧拼成汉信码；WAV 反相声道相加后暴露摩尔斯码；DeepSound 文件中的两首诗构成反切索引表；最后利用小文件的密文完整进入 `$zip2$` hash 这一性质直接恢复数据。

复现时最容易出错的是三点：不要把可变的 ISOBMFF box size 当成已知明文；反切结果要按原始 UTF-8 文本计算 MD5，不能附带换行；最终 64 位十六进制串是直接 AES key，不需要再做 PBKDF2 或当作 ZIP 密码。这样可避免在每一层都误入无意义的口令爆破。
