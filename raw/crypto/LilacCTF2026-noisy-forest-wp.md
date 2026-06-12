# Noisy Forest

## 题目简述

题目对一段长中文文本逐字加入 Unicode 噪声。源码中只有中文字符会消耗随机比特；每个中文字符取 `NOISE_BITS = 1` bit，如果该 bit 为 `1`，字符码点增加 `step_val = 9997`：

```python
huge_int = random.getrandbits(50000)
chunk_val = temp_val & 0xFFFFFFFF
full_bit_stream += format(chunk_val, "032b")
...
noisy_text += chr(ord(char) + noise_val * step)
```

最终 flag 是原文 `input_str` 的 SHA-256：

```python
flag_hash = hashlib.sha256(input_str.encode("utf-8")).hexdigest()
```

因此目标不是恢复某个短密钥，而是尽可能完整恢复原中文长文本，再计算哈希。

## 解题过程

### 关键观察

每个中文字符只有两种可能：

1. 原字符没有变化。
2. 原字符码点加上 `9997`。

这使得单字符候选空间很小。对常见中文字符，可以先尝试 `c` 和 `c - 9997`，再用中文词频表过滤掉明显不合理的生僻字或乱码。官方给出的词频辅助数据是一段按常见程度排列的中文字符表 `hanzipinlv`，可用于第一轮过滤。

### 恢复 MT19937 状态

Python `random.getrandbits(50000)` 底层使用 MT19937。只要从明文/密文差异中恢复出连续 `624 * 32` bit，就能克隆随机数状态。

求解脚本流程如下：

1. 对密文前半部分逐字生成候选：`char` 或 `char - 9997`。
2. 结合标点切分句子，用中文频率和语言模型选出更自然的句子。
3. 将恢复出的前缀与密文比较，提取噪声 bit：

```python
diff = ord(noisy_char) - ord(restored_char)
bit = diff // 9997
```

4. 把前 `624 * 32` bit 按 32 bit 分块喂给 `MT19937Predictor`。
5. 用预测出的后续随机 bit 解密剩余文本。
6. 对完整恢复文本计算 `sha256` 得到 flag。

### 验证

恢复出的完整文本应是连贯中文故事；脚本最后计算：

```python
hashlib.sha256(final_text.encode("utf-8")).hexdigest()
```

并输出 `LilacCTF{<hash>}`。如果前缀恢复中存在错字，后续 MT 状态会错位，最终文本会快速变成乱码，因此文本连贯性本身也是重要校验。

## 方法总结

- 核心技巧：利用低噪声 Unicode 扰动恢复明文前缀，再从明密文差异中克隆 MT19937 状态。
- 识别信号：Python `random.getrandbits` 产生长 bitstream，且密文每个字符泄露一个随机 bit 的取值。
- 复用要点：MT19937 只要拿到连续 624 个 32-bit 输出即可预测后续；当输出被逐 bit 混入明文时，可以先用自然语言冗余恢复一段明文，再反推出随机流。
