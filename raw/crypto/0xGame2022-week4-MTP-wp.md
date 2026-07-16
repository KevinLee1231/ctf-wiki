# week4MTP

## 题目简述

题目把一段英文按 flag 长度切成多个分组，并让每个分组都与同一个 flag 逐字节异或，再输出十六进制密文。也就是说，flag 本身被重复用作一次性密码本的密钥，需要通过 Many-Time Pad 攻击恢复它。

## 解题过程

若两个明文 $M_i,M_j$ 使用同一密钥 $K$，则

$$
C_i\oplus C_j=(M_i\oplus K)\oplus(M_j\oplus K)=M_i\oplus M_j.
$$

密钥被完全消去。英文文本中空格频率很高，而空格 `0x20` 与英文字母异或后通常会得到大小写互换的另一个字母。因此，对某一列尝试每条密文：若它与其他密文在该列异或后大量落入字母范围，则对应明文在该位置很可能是空格。由 $K=C_i\oplus0x20$ 就能得到该列密钥候选。

源码还有一个细节：最后一个明文块不足 flag 长度，输出时用字符 `f` 补齐十六进制行。补出的 `ff` 不是实际密文，不能参与统计，所以先排除最后一行。自动统计后只有少数没有空格样本的列需要根据已经恢复的英文上下文修正。

~~~python
from pathlib import Path


def is_ascii_letter(value):
    return 65 <= value <= 90 or 97 <= value <= 122


lines = Path("output.txt").read_text(encoding="ascii").splitlines()
ciphertexts = [bytes.fromhex(line) for line in lines[:-1]]
width = len(ciphertexts[0])

key = bytearray(width)
for pos in range(width):
    def score(index):
        return sum(
            is_ascii_letter(ciphertexts[index][pos] ^ ciphertexts[other][pos])
            for other in range(len(ciphertexts))
            if other != index
        )

    space_row = max(range(len(ciphertexts)), key=score)
    key[pos] = ciphertexts[space_row][pos] ^ ord(" ")

# 自动结果已显出英文句子，据上下文补三个无可靠空格样本的位置：
# 第 1 行为 "rm cryptosystem is used"；
# 第 4 行为 "tographic system\". A cr"；
# 第 0 行为 "In this meaning, the te"。
cribs = {
    (1, 6): ord("p"),
    (4, 1): ord("o"),
    (0, 10): ord("a"),
}
for (row, pos), plain_byte in cribs.items():
    key[pos] = ciphertexts[row][pos] ^ plain_byte

for ciphertext in ciphertexts:
    plaintext = bytes(c ^ k for c, k in zip(ciphertext, key))
    print(plaintext.decode())

print(bytes(key).decode())
~~~

恢复出的密钥就是 flag：

~~~text
0xGame{ins3cur3|system}
~~~

## 方法总结

一次性密码本的安全前提是密钥只使用一次。重复密钥会让任意两条密文异或后直接暴露明文之间的关系；对自然语言，可利用空格与字母的统计特征恢复大多数密钥字节，再用少量可靠单词作 crib 修正剩余位置。处理输出时还要区分真实密文与格式化填充值，避免污染统计。
