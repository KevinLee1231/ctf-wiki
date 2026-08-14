# multi-xor

## 题目简述

附件给出多行十六进制密文。所有明文都使用同一条一次性密码本进行异或，其中前若干行是自然英语，最后一行包含 Flag。一次性密码本只有在密钥只使用一次时才具有信息论安全性；密钥复用会消去密钥：

$$
C_i\oplus C_j=(P_i\oplus K)\oplus(P_j\oplus K)=P_i\oplus P_j.
$$

## 解题过程

先把每行十六进制字符串还原为字节串，再对任意两行密文做异或。若某个位置的结果是英文字母，很可能其中一条明文在该位置是空格，因为 ASCII 空格 `0x20` 与字母异或后会切换大小写区间。对多组密文统计后即可提出候选密钥字节：

```python
def xor(a, b):
    return bytes(x ^ y for x, y in zip(a, b))

def likely_letter(x):
    return 65 <= x <= 90 or 97 <= x <= 122

cts = [bytes.fromhex(line.strip()) for line in open("cts.txt")]
key = [None] * max(map(len, cts))

for i, ci in enumerate(cts):
    votes = [0] * len(ci)
    for j, cj in enumerate(cts):
        if i == j:
            continue
        for pos, value in enumerate(xor(ci, cj)):
            if likely_letter(value):
                votes[pos] += 1
    for pos, count in enumerate(votes):
        if count >= len(cts) // 2:
            key[pos] = ci[pos] ^ 0x20
```

使用候选密钥解出各行后，根据英语单词、标点和已知 Flag 前缀 `greyhats{` 进行 crib dragging，逐步修正仍有歧义的密钥字节。这里外部 MTP 工具只是自动化上述“密文两两异或、空格投票、人工补全”流程，核心并不依赖该工具本身。

最终最后一行恢复为：

```text
greyhats{n3V3R_Us3_0n3_t1m3_P4D_m0r3_7h4n_0nc3}
```

## 方法总结

- 核心技巧：利用多次使用同一 XOR 密钥后 $C_i\oplus C_j=P_i\oplus P_j$ 的关系进行统计和拖曳猜词。
- 识别信号：多条等长或近似等长密文、自然语言明文、题目强调 one-time pad 被重复使用。
- 复用要点：空格启发式只能产生候选，应使用多条密文交叉验证，并通过已知格式或语义修正错误字节。
