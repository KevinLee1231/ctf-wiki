# BeepBeep

## 题目简述

附件是 8 kHz、16 位的 WAV 音频。开头有 26 个逐级升高的短音，之后是同一组频率组成的长消息。前 26 个音不是正文，而是按顺序给出 `a` 到 `z` 的频率字典；后续每个约 75 ms 的音符用该字典编码一个小写字母。题目的主障碍是从音频载体恢复并按频率重组隐藏符号，故归入 stego，而不是普通文本编码题。

## 解题过程

将单声道 PCM 样本按 75 ms 分块，每块约有 $8000 \times 0.075 = 600$ 个采样。对每块加窗并作 FFT，取正频谱幅值最大的 bin 作为该块的主频。前 26 块依次建立映射：

```python
alphabet = "abcdefghijklmnopqrstuvwxyz"
mapping = {}
for index, chunk in enumerate(chunks[:26]):
    mapping[round(dominant_frequency(chunk, sample_rate))] = alphabet[index]

decoded = "".join(
    mapping.get(round(dominant_frequency(chunk, sample_rate)), "?")
    for chunk in chunks[26:]
)
```

其中 `dominant_frequency` 应先乘 Hann 窗，再从 FFT 的前半段取最大幅值，以减少频谱泄漏造成的相邻频率误判。解出的长文本没有空格；在 Project Nightingale 段落中可找到连续子串：

```text
ductfopenbracketiforonewelcomeouraioverlordsclosebracket
```

将 `openbracket` 和 `closebracket` 还原为花括号后，得到：

```text
DUCTF{iforonewelcomeouraioverlords}
```

若尾部存在少量 `?`，通常是分块边界或最后一段静音造成；只要开头 26 个字典和包含 flag 的连续片段可读，就无需依赖整段长文本的逐字符完全无误。

## 方法总结

- 核心技巧：把已知顺序的前导音当作自描述码表，再以主频解码后续音符。
- 识别信号：音频出现固定时长、离散且重复的单音，且开头有一段数量接近字母表的有序频率时，应优先检查频率符号编码。
- 复用要点：保留原始采样率，按符号时长稳定分块；加窗、频率取整和对尾段容错比盲目调速更可靠。
