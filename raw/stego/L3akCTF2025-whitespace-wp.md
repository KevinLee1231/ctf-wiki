# L3akCTF 2025 Whitespace Writeup

## 题目简述

题目原本用 3×5 ASCII 字体渲染 flag，但随后删除了所有空格。附件中每 6 行表示两个字符：第一行为空行，后 5 行是两个字形同一行的 `#` 被直接拼在一起后的结果。

例如保留空格时，一对字符可能是：

```text
#   ###
#     #
#    ##
#     #
### ###
```

删除空格后只剩：

```text
####
##
###
##
######
```

字符分界和左右顺序都消失了，但每一行 `#` 的数量仍然保留。附件还给出完整字体表及最终 flag 的 MD5：

```text
a7bf5f833c3e4ceff2e006ff801ec16b
```

恢复过程取决于被删除的空间布局和字形指纹，因此本文归入 stego。

## 解题过程

### 为单字符建立五维指纹

对字体表中的每个字符，统计 5 行各自包含多少个 `#`。若字符 $c$ 的指纹记为：

$$
F(c)=(n_0,n_1,n_2,n_3,n_4)
$$

那么两个并排字符 $a,b$ 删除空格后的行计数就是：

$$
F(a,b)=F(a)+F(b)
$$

这个加法会丢失顺序，因此 `ab` 与 `ba` 的指纹相同；不同字形也可能产生相同计数组合。单个块通常对应多个候选字符对。

### 解析附件并生成候选对

附件共有 252 行，即 $84/2\times 6$，所以 flag 长度为 84。下面的代码读入字体表和被破坏的 flag，为 42 个位置分别生成所有候选字符对：

```python
from collections import defaultdict

ALPHABET = (
    "abcdefghijklmnopqrstuvwxyz"
    "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    "0123456789!_{}"
)
FLAG_LENGTH = 84


def load_fingerprints(path):
    rows = open(path, encoding="utf-8").readlines()[1:]
    fingerprints = {}

    for index, char in enumerate(ALPHABET):
        glyph = [row[index * 4:(index + 1) * 4] for row in rows]
        fingerprints[char] = tuple(line.count("#") for line in glyph)

    return fingerprints


def load_blocks(path):
    rows = open(path, encoding="utf-8").readlines()
    blocks = []

    for index in range(FLAG_LENGTH // 2):
        glyph_rows = rows[index * 6 + 1:(index + 1) * 6]
        blocks.append(tuple(line.count("#") for line in glyph_rows))

    return blocks


fingerprints = load_fingerprints("alphabet.txt")
blocks = load_blocks("flag.txt")
chunks = []

for observed in blocks:
    candidates = set()
    for left in ALPHABET:
        for right in ALPHABET:
            combined = tuple(
                a + b
                for a, b in zip(fingerprints[left], fingerprints[right])
            )
            if combined == observed:
                candidates.add(left + right)
    chunks.append(candidates)
```

### 用格式和语言结构逐步收窄

直接计算 42 个候选集合的笛卡尔积，组合数约为：

```text
61775263599091771244544000
```

先固定 CTF flag 的已知格式：

```python
def force_substring(index, text):
    if index < 0:
        index = FLAG_LENGTH + index - len(text) + 1

    for offset, expected in enumerate(text):
        chunk_index = (index + offset) // 2
        side = (index + offset) % 2
        chunks[chunk_index] = {
            pair
            for pair in chunks[chunk_index]
            if pair[side] == expected
        }


force_substring(0, "L3AK{")
force_substring(-1, "}")
```

这一步仍有约 $2.01\times 10^{22}$ 种组合。继续利用 flag 正文只使用小写字母、数字和下划线的特征过滤：

```python
for index in range(2, len(chunks)):
    chunks[index] = {
        pair
        for pair in chunks[index]
        if all(char.islower() or char.isdigit() or char in "_{}" for char in pair)
    }
```

由于相邻两个字符共享一个候选对，人工从末尾观察候选比从头全局爆破有效得多。末尾可以依次辨认出：

```text
d0wn}
_1t_d0wn}
_t0_n4rr0w_1t_d0wn}
_4_h3ur1st1c_t0_n4rr0w_1t_d0wn}
```

每确认一段就调用 `force_substring`，再重新查看前一个词的候选。沿下划线分词从右向左推进，可恢复完整的英语式 leet 文本。MD5 只用于最终消歧和防止人工抄错，而不是在最初的巨大空间上盲目爆破。

### 校验结果

恢复出的候选为：

```text
L3AK{jus7_p4tt3rn_m4tch1ng_4t_f1rs7_bu7_th3n_y0u_n33d_4_h3ur1st1c_t0_n4rr0w_1t_d0wn}
```

最终校验：

```python
from hashlib import md5

flag = (
    "L3AK{jus7_p4tt3rn_m4tch1ng_4t_f1rs7_bu7_th3n_"
    "y0u_n33d_4_h3ur1st1c_t0_n4rr0w_1t_d0wn}"
)

assert len(flag) == 84
assert md5(flag.encode()).hexdigest() == "a7bf5f833c3e4ceff2e006ff801ec16b"
print(flag)
```

## 方法总结

删除空格没有完全销毁 ASCII 字形信息，而是把每行像素压缩成了两个字符的计数和。本题先把视觉问题转成五维向量匹配，再使用 flag 格式、字符集和自然语言分词逐层消除歧义。

这类信息丢失题不应一开始就枚举完整明文。更有效的做法是先确定仍然保留的不变量，本题是不受空格删除影响的行内 `#` 数；再按局部块建立候选集合，最后利用跨块结构和强校验值完成全局恢复。
