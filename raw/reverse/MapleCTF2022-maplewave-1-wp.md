# maplewave-1

## 题目简述

`maplewave-1` 与前题共用容器，但 codec 字节为 1。负载按 128 个采样一帧保存，先对原始 unsigned 8-bit PCM 做帧内差分，再用带游程标记的有符号指数 Golomb 编码压缩。需要准确复现位流读取，而不是把负载直接当 PCM。

## 解题过程

从偏移 `0x10` 起按最高位优先读位。一个无符号数先读取连续的 1，直到终止 0；1 的数量决定阶数，再读取相应尾数，并补上隐含最高位。有符号值在前面再带一位符号：

```python
def read_signed():
    sign = read_bit()
    value = read_unsigned_golomb()
    if sign and value == 0:
        return RLE_MARKER
    return -value if sign else value
```

“负零”专门表示 RLE，后面再读一个无符号数作为重复次数，重复上一差分值。每帧解出 128 个差分后执行 `cumsum`，并按 `uint8` 回绕；下一帧重新从该帧的差分序列累计：

```python
for _ in range(frame_count):
    delta = decode_rle(128)
    samples.extend(np.cumsum(delta, dtype=np.uint8))
```

写成 1 通道、8-bit、16 kHz WAV 后，语音为 “lossless difference encoding”，末尾数字为 3、6、0、4：

```text
maple{lossless difference encoding 3604}
```

## 方法总结

压缩格式逆向要严格记录位序、终止位、符号映射和 RLE 哨兵。本题的负零只有在保留独立符号位时才可能出现，是识别游程标记的关键。差分积分是每 128 个采样重新开始，若错误地跨帧累计，音频会逐帧产生明显直流漂移。
