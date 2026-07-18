# week1CuteCaesar

## 题目简述

附件由 `嗷`、`呜`、`啊`、`~` 等字符组成，需要先按兽语编码还原，再根据题名进行凯撒解密。

## 解题过程

第一层不是自然语言，而是“兽音译者”格式：固定头为 `~呜嗷`、结尾为 `啊`，主体字符表按 `嗷呜啊~` 排列。主体每两个字符合成一个 $0$ 到 $15$ 的数，再减去当前组序号模 $16$；每四个还原出的十六进制半字节组成一个 Unicode 码元。完整本地解码如下：

```python
from pathlib import Path

ciphertext = Path("cipher.txt").read_text(encoding="utf-8").strip()
if not (ciphertext.startswith("~呜嗷") and ciphertext.endswith("啊")):
    raise ValueError("不是预期的兽音译者格式")

alphabet = "嗷呜啊~"
body = ciphertext[3:-1]
if len(body) % 2:
    raise ValueError("兽音主体长度应为偶数")

hex_digits = []
for i in range(0, len(body), 2):
    high = alphabet.index(body[i])
    low = alphabet.index(body[i + 1])
    value = (high * 4 + low - (i // 2) % 16) % 16
    hex_digits.append(format(value, "x"))

hex_text = "".join(hex_digits)
plain = "".join(
    chr(int(hex_text[i:i + 4], 16))
    for i in range(0, len(hex_text), 4)
)
print(plain)
```

运行后得到：

```text
0aJdph{fdhvdu_1v_q0w_fxwh}
```

中间结果保留了类似 flag 的结构，结合题名 `Caesar` 可知第二层是凯撒密码。英文字母向前移动 3 位，数字、下划线和花括号保持不变：

```python
text = "0aJdph{fdhvdu_1v_q0w_fxwh}"

def shift_back(ch):
    if "a" <= ch <= "z":
        return chr((ord(ch) - ord("a") - 3) % 26 + ord("a"))
    if "A" <= ch <= "Z":
        return chr((ord(ch) - ord("A") - 3) % 26 + ord("A"))
    return ch

print("".join(map(shift_back, text)))
```

输出：

```text
0xGame{caesar_1s_n0t_cute}
```

## 方法总结

套娃编码应保留每一层的中间结果，并利用字符集和题名判断下一层。本题不依赖在线解码器：兽语层得到的明文和凯撒位移量都已在正文中给出，第二层可直接本地复现。
