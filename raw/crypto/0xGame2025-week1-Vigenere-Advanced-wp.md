# Vigenere Advanced

## 题目简述

题目仍使用 94 字符扩展字母表和循环密钥，但加密不再是线性偏移，而是对明文索引 $P_i$ 计算：

$$
C_i=(P_i+K_i)P_i\bmod 94
$$

这是模合数下的二次同余，同一密文位置可能对应多个明文字符。题目同时约束 flag 内部只含小写字母，可用于剪枝。

## 解题过程

字母表只有 94 个字符，因此无需实现通用二次同余求解器。对每个密文位置枚举全部 `char_index`，保留满足加密等式的候选；随后对各位置候选集合做笛卡尔积，并用 `0xGame{...}` 格式与小写字符约束筛选。

```python
from string import digits, ascii_letters, punctuation, ascii_lowercase
from itertools import product
from tqdm import tqdm

key = "QAQ(@.@)"
alphabet = digits + ascii_letters + punctuation

def vigenere_decrypt(ciphertext, key):
    plaintext = []
    key_index = 0
    for i in ciphertext:
        tmp = []
        bias = alphabet.index(key[key_index])
        new_index = alphabet.index(i)
        for char_index in range(len(alphabet)):
            if ((char_index + bias) * char_index) % len(alphabet) == new_index:
                tmp.append(alphabet[char_index])
        plaintext.append(tmp)
        key_index = (key_index + 1) % len(key)
    return plaintext

ciphertext = "0l0CSoYM<c;amo_P_"

tmp = vigenere_decrypt(ciphertext, key)
total_num = 1
for i in tmp:
    total_num *= len(i)
print(f"Total number of combinations: {total_num}")

for i in tqdm(product(*tmp), total=total_num):
    flag = ''.join(i)
    if flag.startswith("0xGame{") and flag.endswith("}") and set(flag[7:-1]) < set(ascii_lowercase):
        print(flag)
```

得到输出：

```text
0xGame{axcellent}
0xGame{axcellezt}
0xGame{axcsllent}
0xGame{axcsllezt}
0xGame{excellent}
0xGame{excellezt}
0xGame{excsllent}
0xGame{excsllezt}
```

很容易猜测 flag 内容时一个可读的单词，所以是 `0xGame{excellent}`。

多个候选都满足模方程，说明数学约束本身不足以唯一确定明文；最终利用题目给出的字符集约束和可读性选出 `excellent`，而不是任意取每个位置的第一个根。

## 方法总结

- 核心技巧：逐位置枚举模二次方程的所有根，再组合并施加格式约束。
- 识别信号：非线性索引变换、模数为合数、逆映射一对多，以及题目对明文字符集的额外限制。
- 复用要点：解密函数应返回候选集合而不是单个字符；组合数量较大时，尽早利用已知前后缀、字符集和语言可读性剪枝。
