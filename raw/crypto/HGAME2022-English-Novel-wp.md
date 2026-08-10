# English Novel

## 题目简述

附件包含被打乱顺序的英文小说明文片段、对应的 Vigenere 式密文片段、加密脚本和 `flag.enc`。加密只移动英文字母并保留大小写，空格、标点和数字完全不变，因此可以先借助非字母位置匹配明密文文件，再实施已知明文攻击恢复逐位置密钥。

## 解题过程

先把每个字母替换成统一占位符，保留所有非字母字符，构造不会被加密改变的结构特征：

```python
def feature(text):
    return "".join("*" if char.isalpha() else char for char in text)
```

对 `original` 与 `encrypt` 两个目录分别按该特征分组。若某个特征在两边都只出现一次，就能确定一对明密文。对于字母位置，密钥满足：

$$
k_i=(C_i-P_i)\bmod 26
$$

非字母位置不参与位移，可以继续从其他唯一配对中补齐尚未确定的密钥槽。原 PDF 给出的自动匹配思路可整理为：

```python
from collections import defaultdict
from pathlib import Path

def feature(text):
    return "".join("*" if char.isalpha() else char for char in text)

def letter_value(char):
    return ord(char.lower()) - ord("a")

pairs = defaultdict(lambda: {"plain": [], "cipher": []})

for path in Path("original").rglob("*"):
    if path.is_file():
        text = path.read_text()
        pairs[feature(text)]["plain"].append(text)

for path in Path("encrypt").rglob("*"):
    if path.is_file():
        text = path.read_text()
        pairs[feature(text)]["cipher"].append(text)

flag_ciphertext = Path("flag.enc").read_text()
key = [None] * len(flag_ciphertext)

for group in pairs.values():
    if len(group["plain"]) != 1 or len(group["cipher"]) != 1:
        continue
    plain = group["plain"][0]
    cipher = group["cipher"][0]
    for index, (p_char, c_char) in enumerate(zip(plain, cipher)):
        if index >= len(key):
            break
        if p_char.isalpha():
            key[index] = (letter_value(c_char) - letter_value(p_char)) % 26
```

多组唯一配对补全后，前 51 位密钥为：

```python
key = [
    16, 20, 0, 10, 13, 3, 24, 7, 24, 5, 22, 17, 17,
    18, 5, 13, 16, 14, 9, 5, 2, 15, 16, 0, 20, 25,
    18, 13, 7, 8, 22, 22, 4, 0, 8, 3, 11, 23, 25,
    0, 6, 3, 3, 10, 2, 8, 0, 5, 13, 24, 0,
]
```

解密 `flag.enc` 时只对字母做逆向位移：

```python
def decrypt(text, shifts):
    output = []
    for index, char in enumerate(text):
        if char.isupper():
            output.append(chr((ord(char) - ord("A") - shifts[index]) % 26 + ord("A")))
        elif char.islower():
            output.append(chr((ord(char) - ord("a") - shifts[index]) % 26 + ord("a")))
        else:
            output.append(char)
    return "".join(output)

ciphertext = "xaawr{B0_d0l_cs0m_'Pp0mn-odn1vpabt_deqzcq'?}"
print(decrypt(ciphertext, key))
```

最终得到：

```text
hgame{D0_y0u_kn0w_'Kn0wn-pla1ntext_attack'?}
```

完整密钥和 `flag.enc` 内容通过 [上辰的已知明文攻击记录](https://www.cnblogs.com/sCh3n/p/15917384.html) 核对；匹配依据、密钥公式和解密代码均已写入正文。

## 方法总结

乱序并不能阻止已知明文攻击，因为加密保留了空格与标点布局。先用不变量建立明密文对应关系，再从多个唯一配对补齐密钥，能避免仅凭文件名或序号错误配对。恢复时还要区分大小写基准，并保证非字母位置不被移位。
