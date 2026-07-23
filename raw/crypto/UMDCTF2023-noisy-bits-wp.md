# Noisy Bits

## 题目简述

生成器把 360 位 flag 切成 30 个 12 位消息块，使用多项式 `0xC75` 编码为 23 位码字，然后对每个码字随机翻转三次比特。题面中的 “Go lay” 和 23 位码长共同指向二元 Golay
$(23,12,7)$ 码。

该码的最小距离为 7，能够唯一纠正最多
$\lfloor(7-1)/2\rfloor=3$ 个错误，正好覆盖生成器加入的噪声上限。

## 解题过程

### 复现系统编码器

官方 `encode()` 保留低 12 位原消息，并通过生成多项式计算高位校验部分：

```python
POLY = 0xC75

def encode(message):
    work = message & 0xFFF
    original = work
    for _ in range(12):
        if work & 1:
            work ^= POLY
        work >>= 1
    return (work << 12) | original
```

因为消息空间只有 $2^{12}=4096$ 个元素，可以预先生成全部合法码字，不必实现更复杂的综合译码。

### 以 Hamming 距离纠错

对输出中的每个 23 位受损码字 $r$，枚举合法码字 $c$，寻找

$$
d_H(r,c)\le3
$$

的唯一候选：

```python
valid = {encode(m): m for m in range(1 << 12)}
decoded_blocks = []

for received in received_words:
    matches = [
        message
        for codeword, message in valid.items()
        if (received ^ codeword).bit_count() <= 3
    ]
    assert len(matches) == 1
    decoded_blocks.append(f"{matches[0]:012b}")

bits = "".join(decoded_blocks)
flag = int(bits, 2).to_bytes(len(bits) // 8, "big")
print(flag.decode())
```

这里用异或结果的置位数就是 Hamming 距离。即使生成器三次随机翻转碰巧选中同一位置，实际错误数只会减少，不会超过纠错半径。

解码结果为：

```text
UMDCTF{n0i5y_ch4nn3l5_ar3_n0t_@_pr0bleM_4_u}
```

## 方法总结

- 核心技巧：识别 Golay $(23,12,7)$ 纠错码，利用其三位纠错能力恢复每个被扰动的码字。
- 识别信号：12 位输入、23 位输出、固定多项式和每块至多三位噪声是非常强的编码参数特征。
- 复用要点：小消息空间时穷举码本既直观又可靠；务必断言每块只有一个候选，并按原始块顺序拼接固定 12 位结果，避免丢失前导零。
