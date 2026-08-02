# drm-2

## 题目简述

题目直接给出 DRM key、nonce 和加密歌曲，却提示客户端没有播放歌曲中的某一部分。客户端把音频按每块 44100 个 16 位采样拆分，按硬编码 `order` 取块，用一个连续的 ChaCha20 状态依次解密；解密后首采样为 0 的块会被静音跳过，而这些正是含 flag 信号的块。

## 解题过程

先从客户端源码、反编译结果或官方 solver 取得完整 `order` 数组。加密端对第 $i$ 个顺序项消耗下一段 ChaCha20 密钥流，却把密文存到

$$
\text{new\_b}=\left\lfloor\frac{\text{order}[i]}{44100}\right\rfloor
$$

对应的位置。因此解密时必须按 `order` 的迭代顺序取出这些存储块，并持续复用同一个 ChaCha20 对象：

```python
from Crypto.Cipher import ChaCha20

block_samples = 88100 // 2
block_bytes = block_samples * 2
data = open("enc_song.dat", "rb").read()
blocks = [data[i:i + block_bytes] for i in range(0, len(data), block_bytes)]

cipher = ChaCha20.new(
    key=open("drm.key", "rb").read(),
    nonce=open("drm.nonce", "rb").read(),
)

song = bytearray()
for offset in order:
    stored_index = offset // block_samples
    song.extend(cipher.decrypt(blocks[stored_index]))
open("song.dat", "wb").write(song)
```

不要像原客户端那样过滤首采样为 0 的块。播放完整恢复结果后，隐藏段是摩尔斯电码，解码并按题面要求使用小写，得到：

```text
tjctf{idon'tliketosing1549089769}
```

## 方法总结

- 流密码状态与块的逻辑顺序绑定；即使密文被打乱存储，也必须按加密时消耗密钥流的顺序解密。
- 客户端的“首采样非零才播放”不是数据完整性检查，而是主动隐藏特定块的显示逻辑。
- 这道题的主障碍是还原客户端块映射和 ChaCha20 状态机；音频摩尔斯只是重建后的最后一层编码。
