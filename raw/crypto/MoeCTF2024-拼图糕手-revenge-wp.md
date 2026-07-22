# 拼图糕手（revenge）

## 题目简述

附件是 16 张未旋转的二维码切片。拼成 $4\times4$ 二维码并扫描后，得到一段 `encoded_flag{...}`；真正的主要障碍是按附件 `encode.py` 的相反顺序撤销字符串反转、自定义字母置换、数字链式异或和特殊符号代换，因此归入 Crypto。

## 解题过程

先利用三个定位块、两条时序线和相邻边连续性排列切片，整个过程不需要旋转。扫码文本中的有效部分为：

```text
encoded_flag{71517ysd%ryxsc!usv@ucy*wqosy*qxl&sxl*sbys^wb$syqwp$ysyw!qpw@hs}
```

`encode.py` 的顺序并不是简单逐字符替换：

1. 小写字母按三个区间置换；
2. 原文数字暂时跳过，最后把相邻数字异或值与末位数字追加到结尾；
3. 去掉变换结果中的空格；
4. 把整个字符串反转。

所以解密必须先反转密文，再把末尾的数字串分离出来，最后撤销字母置换和数字链。原稿 `decode.py` 误用了未定义的 `input_text`，也再次执行了数字正向异或；下面是针对本题实例的完整逆变换：

```python
import re

encoded = (
    "71517ysd%ryxsc!usv@ucy*wqosy*qxl&sxl*sbys^"
    "wb$syqwp$ysyw!qpw@hs"
)

def encode_lower(char):
    value = ord(char)
    if value < ord("h"):
        return chr(value + 19)
    if value > ord("s"):
        return chr(value - 19)
    return chr(219 - value)

inverse_lower = {
    encode_lower(char): char
    for char in "abcdefghijklmnopqrstuvwxyz"
}

# 撤销最外层 reverse_encoding。
stream = encoded[::-1]
match = re.fullmatch(r"(.*?)(\d+)", stream)
letter_part, digit_code = match.groups()

decoded = "".join(inverse_lower.get(char, char) for char in letter_part)

# 正向第 i 项为 d[i] ^ d[i + 1]（0 <= i < n - 1），最后一项为 d[n - 1]。
# 本题每个异或结果都只有一位十进制数，因此可逐字符解析。
digits = [int(digit_code[-1])]
for value in reversed(digit_code[:-1]):
    digits.append(int(value) ^ digits[-1])
decoded += "".join(map(str, reversed(digits)))

# Hint 的含义是特殊字符由同一数字键上的数字替代。
decoded = decoded.translate(str.maketrans("!@#$%^&*", "12345678"))
print(f"moectf{{{decoded}}}")
```

数字部分的逆推为：

```text
71517 -> 52367
```

附件给出的 Base64 提示还需经过同一套反向字母置换，含义是 `StrangeCharacterStaywithNumberOnSomewhere`；结合补充 Hint，即把 `!@#$%^&*` 替换为键盘同键的 `12345678`。最终输出：

```text
moectf{hs2dkj1dfhf4kdjfh4ud6hfuh8oeh7oej8fhljd8fvb2chb1vhefi5whf52367}
```

## 方法总结

这题最容易错在变换顺序。数字没有在原位置编码，而是集中追加后再随整串反转；解密数字链时也必须从已知末位向前递推。二维码只负责承载密文，自定义编码才是决定最终 flag 的步骤。
