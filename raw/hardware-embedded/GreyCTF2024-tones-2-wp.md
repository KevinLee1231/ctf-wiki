# GreyCTF2024 Tones 2 WP

## 题目简述

附件只使用 300 Hz 和 800 Hz 两个 FSK 音调，每个符号长 1200 个采样点。每七个符号是一组 Hamming(7,4) 码，并以 80% 概率随机翻转其中一位。生成器还有一个关键缺陷：每个字符只写入低四位的编码，高四位虽然计算了却没有追加到音频。

## 解题过程

先按固定符号长度分块，用 FFT 峰值把 300 Hz 映射为 0、800 Hz 映射为 1。对每七位 $c_1,\ldots,c_7$ 计算校验综合：

$$
s_1=c_1\oplus c_3\oplus c_5\oplus c_7
$$

$$
s_2=c_2\oplus c_3\oplus c_6\oplus c_7
$$

$$
s_4=c_4\oplus c_5\oplus c_6\oplus c_7
$$

错误位置为 $s_1+2s_2+4s_4$；非零时翻转对应位，再取位置 3、5、6、7 得到四个数据位：

```python
def decode_hamming(c):
    s1 = c[0] ^ c[2] ^ c[4] ^ c[6]
    s2 = c[1] ^ c[2] ^ c[5] ^ c[6]
    s4 = c[3] ^ c[4] ^ c[5] ^ c[6]
    bad = s1 | (s2 << 1) | (s4 << 2)
    if bad:
        c[bad - 1] ^= 1
    data = [c[2], c[4], c[5], c[6]]
    return sum(bit << i for i, bit in enumerate(data))
```

这样只能恢复每个字符的低半字节。审查官方生成器可以确认，高半字节的 `hamming_bits` 在第二次计算后没有执行 `symbols += hamming_bits`，所以附件本身信息不足，无法唯一确定完整 ASCII。结合已知 `grey{...}` 格式、可读短语以及仓库给出的官方答案，完整 flag 为：

```text
grey{why_th3_h4mm1ng_c0d3_4398th489h94isdgodshf}
```

## 方法总结

Hamming(7,4) 能纠正单比特错误，却不能恢复从未发送的数据。正确题解必须区分“纠错成功”和“信息完整”：本题音频只确定低半字节，完整答案还依赖格式与官方生成材料。把这一限制写清楚比假装附件可唯一解更严谨。
