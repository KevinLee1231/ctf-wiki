# strange_image_plus+

## 题目简述

服务把一张 $72\times60$ 的 RGB 黑白图像按用户指定的 `chunk_size` 分组。外层结构类似 CBC：当前明文分组先与上一分组密文（首组使用 IV）异或，再交给流加密函数；流加密会把每个字节拆成两个半字节，分别经过 3 轮 S 盒与 LFSR 输出异或。

```python
def encryption(self, m_raw):
    c_raw = b""
    k = self.iv
    for i in range(0, len(m_raw), self.chunk_size):
        m = m_raw[i:i + self.chunk_size]
        k = self.stream_encryption(xor_bytes(m, k))
        c_raw += k
    return c_raw
```

flag 图像只包含黑、白两种像素，每个像素占 3 字节。服务允许用户控制 LFSR taps、IV、分组大小和上传图像，因此核心不是图像隐写，而是利用可控参数制造极短 LFSR 周期，再从 CBC 分组关系恢复黑白像素。

## 解题过程

### 预期解：构造 9 周期并逐块恢复

LFSR 输出会周期性重复。若把 8 个寄存器全部放入反馈，周期可固定为 9；令 `chunk_size=18`，便能让一个分组的前、后 9 字节使用相同的密钥流。再设置全零 IV，并上传全黑图片作为已知输入。

在密钥流重复的条件下，正常分组的前后两半会保持相同。遇到前后半不一致的首个分组，差异位置就对应 flag 图像中需要判断的像素。由于每个像素只能是 `0x000000` 或 `0xffffff`，一个半分组包含 3 个像素，只需枚举 $2^3=8$ 种组合。

核心恢复过程如下；`send_img` 负责以相同 taps、IV 和分组大小向服务提交候选图像，并返回密文和非空分组计数：

```python
import itertools

chunk_size = 18
i = 0

while i < len(ciphertext):
    current = ciphertext[i:i + chunk_size]
    different_pixels = []

    for offset in range(0, chunk_size // 2, 3):
        if current[offset] != current[offset + chunk_size // 2]:
            different_pixels.append(offset // 3)

    if different_pixels:
        for candidate in itertools.product([b"\x00", b"\xff"], repeat=3):
            first_half = b""
            second_half = b""

            for pixel_index, value in enumerate(candidate):
                first_half += value * 3
                if pixel_index in different_pixels:
                    second_half += (
                        b"\x00" if value == b"\xff" else b"\xff"
                    ) * 3
                else:
                    second_half += value * 3

            trial = (
                image_bytes[:i]
                + first_half
                + second_half
                + image_bytes[i + chunk_size:]
            )
            new_ciphertext, new_count = send_img(trial)
            if new_count < non_empty_count:
                image_bytes = trial
                ciphertext = new_ciphertext
                non_empty_count = new_count
                break

    i += chunk_size
```

当候选使非空分组数量下降时，说明该候选与 flag 对应分组互补，可以接受并继续处理下一块。全部分组完成后，把恢复的字节按 RGB 三元组写回图像即可读取 flag。

### 非预期解：重复 taps 令密钥周期退化为 0

服务只检查了某些会让 LFSR 后半状态全零的非法情况，却没有检查 taps 是否重复。因此可以把 8 个 taps 全部设为同一个位置，例如：

```python
taps_list = [[0] * 8] * 4
```

重复异或同一位偶数次后反馈恒为 0，LFSR 密钥序列随即退化成全零周期。此时无需逐像素枚举，直接按外层 CBC 关系逆推各分组即可恢复原始 flag 图像。这条路径比预期解更短，但依赖参数校验缺陷；预期解则展示了即使 taps 不完全退化，短周期仍会泄露明文结构。

## 方法总结

- 核心技巧：主动选择 LFSR taps 和分组大小，使同一密钥流在一个 CBC 分组内部重复，再利用黑白像素的低熵逐块恢复明文。
- 识别信号：可控 IV、可控分组大小、可控 LFSR 参数，以及“图像只有两种像素值”的组合会把密码分析降为很小的枚举问题。
- 复用要点：除理论周期外，还要检查参数验证是否拒绝重复 taps、零反馈和短循环；实现层退化往往比预期密码分析更直接。
