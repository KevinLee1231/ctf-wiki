# Ex Viginere?

## 题目简述

本题把普通 Vigenère 的“周期加法”改成周期仿射变换。对只含小写字母的明文，令字母编号为 $0$ 到 $25$，加密满足

$$
c_i\equiv
m_i\cdot k^{(1)}_{i\bmod11}
+k^{(2)}_{i\bmod7}
\pmod{26}.
$$

乘法密钥的每个元素都与 26 互素，因此各位置的变换可逆。两组密钥长度分别为 11 和 7，组合后的完整周期为

$$
\operatorname{lcm}(11,7)=77.
$$

flag 内容作为一个短字符串嵌在解密后的英文长文本中。题目另给出固定后缀

```text
How_Interesting_the_Cryptography_Is
```

以及 `md5(flag_content || suffix)` 的值，用于从长明文中定位正确子串。当前源码仓库没有保留本题的 `task.py` 和密文附件；下面的算法、摘要和恢复出的密钥来自官方总题解，不补造缺失数据。

## 解题过程

### 1. 用重合指数确定组合周期

对候选周期 $t$，把密文按下标模 $t$ 分列。若 $t$ 是正确周期，每列都只经过同一个仿射替换，字母分布仍保留英文的非均匀性，其重合指数会接近自然英文。

一列长度为 $N$，字母 $j$ 出现 $n_j$ 次时，

$$
IC=\frac{\sum_{j=0}^{25}n_j(n_j-1)}{N(N-1)}.
$$

枚举周期并计算各列平均重合指数，可以观察到 77 是稳定峰值；结合 $77=7\times11$，对应两组互素长度的密钥。

### 2. 逐列恢复仿射参数

固定余数类 $r=i\bmod77$ 后，加密退化为单表仿射密码：

$$
c\equiv am+b\pmod{26}.
$$

枚举满足 $\gcd(a,26)=1$ 的 $a$ 和所有 $b$，按英文频率的卡方值选择最优明文。77 列分别得到参数对 $(a_r,b_r)$ 后：

- $a_r$ 只由 $r\bmod11$ 决定；
- $b_r$ 只由 $r\bmod7$ 决定。

对相同余数类取众数，可消除个别短列的频率误判。下面的代码给出了周期检测和参数恢复主线：

```python
from collections import Counter
from math import gcd
from pathlib import Path

ALPHABET = "abcdefghijklmnopqrstuvwxyz"
ENGLISH = (
    0.08167, 0.01492, 0.02782, 0.04253, 0.12702, 0.02228,
    0.02015, 0.06094, 0.06966, 0.00153, 0.00772, 0.04025,
    0.02406, 0.06749, 0.07507, 0.01929, 0.00095, 0.05987,
    0.06327, 0.09056, 0.02758, 0.00978, 0.02360, 0.00150,
    0.01974, 0.00074,
)

ciphertext = Path("cipher").read_text(encoding="utf-8").strip()


def index_of_coincidence(text):
    if len(text) < 2:
        return 0.0
    counts = Counter(text)
    numerator = sum(value * (value - 1) for value in counts.values())
    return numerator / (len(text) * (len(text) - 1))


def average_ic(text, period):
    columns = [text[offset::period] for offset in range(period)]
    return sum(index_of_coincidence(column) for column in columns) / period


period_scores = [
    (abs(average_ic(ciphertext, period) - 0.065), period)
    for period in range(1, 101)
]
print(sorted(period_scores)[:10])


def decrypt_column(column, multiplier, additive):
    inverse = pow(multiplier, -1, 26)
    return [
        ((ALPHABET.index(ch) - additive) * inverse) % 26
        for ch in column
    ]


def chi_square(values):
    counts = Counter(values)
    length = len(values)
    return sum(
        (counts[index] - length * ENGLISH[index]) ** 2
        / (length * ENGLISH[index])
        for index in range(26)
    )


pairs = []
for offset in range(77):
    column = ciphertext[offset::77]
    candidates = []
    for multiplier in range(26):
        if gcd(multiplier, 26) != 1:
            continue
        for additive in range(26):
            values = decrypt_column(column, multiplier, additive)
            candidates.append((chi_square(values), multiplier, additive))
    _, multiplier, additive = min(candidates)
    pairs.append((multiplier, additive))

multiplier_key = [
    Counter(pairs[r][0] for r in range(77) if r % 11 == index)
    .most_common(1)[0][0]
    for index in range(11)
]
additive_key = [
    Counter(pairs[r][1] for r in range(77) if r % 7 == index)
    .most_common(1)[0][0]
    for index in range(7)
]

print(multiplier_key)
print(additive_key)
```

官方题解恢复出的两组密钥为：

```python
multiplier_key = [9, 7, 25, 11, 11, 19, 5, 3, 25, 9, 7]
additive_key = [25, 19, 18, 19, 25, 20, 8]
```

### 3. 解密并用摘要定位 flag

逐字符执行仿射逆变换：

$$
m_i\equiv
\left(c_i-k^{(2)}_{i\bmod7}\right)
\left(k^{(1)}_{i\bmod11}\right)^{-1}
\pmod{26}.
$$

再枚举明文中的短子串，用题目给定的 MD5 摘要验证：

```python
from hashlib import md5

multiplier_key = [9, 7, 25, 11, 11, 19, 5, 3, 25, 9, 7]
additive_key = [25, 19, 18, 19, 25, 20, 8]

plain = []
for index, char in enumerate(ciphertext):
    value = ALPHABET.index(char)
    multiplier = multiplier_key[index % len(multiplier_key)]
    additive = additive_key[index % len(additive_key)]
    decoded = (value - additive) * pow(multiplier, -1, 26) % 26
    plain.append(ALPHABET[decoded])

plaintext = "".join(plain)

suffix = b"How_Interesting_the_Cryptography_Is"
expected = "196cf7098c7ea6e3e4d03691fb9d4f58"

for length in range(1, 21):
    for start in range(len(plaintext) - length + 1):
        candidate = plaintext[start:start + length]
        if md5(candidate.encode() + suffix).hexdigest() == expected:
            print(f"moectf{{{candidate}}}")
            raise SystemExit
```

最终得到：

```text
moectf{pieceofchocolate}
```

## 方法总结

- 核心技巧：把两个互素周期的仿射密钥合并为长度 77 的单表列，使用重合指数定周期、频率评分恢复每列参数。
- 识别信号：加密同时出现周期乘法和周期加法时，真实统计周期通常是两个长度的最小公倍数，而不是任一单独长度。
- 复用要点：乘法系数必须在模 26 下可逆；长明文恢复后，题目给出的摘要适合做子串定位与结果验证，不能替代前面的密码分析。
