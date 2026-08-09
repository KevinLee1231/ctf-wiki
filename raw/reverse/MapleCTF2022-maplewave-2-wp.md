# maplewave-2

## 题目简述

`maplewave-2` 使用 codec 2，在 128 样本帧上加入离散余弦变换。每帧的 DC 系数跨帧差分编码，127 个 AC 系数使用与前题相同的有符号指数 Golomb 编码和 RLE；解码后还要执行逆 DCT 和幅度反归一化。

## 解题过程

容器头仍为 16 字节。维护跨帧变量 `dc=0`，每帧先读一个有符号 Golomb 值并累加到 DC，再通过 RLE 解出 127 个 AC 系数：

```python
dc = 0
samples = []
for _ in range(frame_count):
    dc += read_signed_golomb()
    ac = decode_rle(127)
    coeff = np.concatenate(([dc], ac))
    block = scipy.fft.idct(coeff)
    block = np.uint8(np.clip(block * 128 + 128, 0, 256))
    samples.extend(block)
```

这里 DC 只有一个跨帧状态，AC 的 RLE 则只负责凑满当前帧；两者不能混用。重建的数组仍按单声道 unsigned 8-bit、16 kHz 写入 WAV。播放后可听到 “discrete cosine transform”，数字依次为 7、3、8、4：

```text
maple{discrete cosine transform 7384}
```

## 方法总结

变换编码的恢复顺序必须与编码完全相反：熵解码、RLE 展开、DC 差分累加、逆 DCT、反归一化。若音频大体可辨但削波严重，应检查 `*128+128` 和 unsigned 8-bit 映射；若每帧边界跳变，则应检查 DC 是否正确跨帧累加。最终语音内容也概括了本级所用的核心变换。
