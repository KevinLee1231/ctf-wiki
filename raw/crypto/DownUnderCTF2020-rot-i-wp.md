# DownUnderCTF 2020 - rot-i

## 题目简述

附件给出一段保留空格、标点和大小写的英文字母密文，题面提示“ROT13 is boring”。它不是对所有字符使用同一个 Caesar 位移，而是让第 $i$ 个字符按其在整段消息中的索引向前移动 $i$ 位；非字母同样占据索引，但不会被替换。

## 解题过程

密文开头是：

```text
Ypw'zj zwufpp hwu txadjkcq dtbtyu kqkwxrbvu!
```

根据英文词形可以猜测第一个词是 `You've`。逐字符核对会发现 `Y` 未移动、`o` 向前移动 1 位得到 `p`、`u` 向前移动 2 位得到 `w`。因此若密文字母为 $c_i$，解密就是在对应大小写字母表中计算：

$$
m_i=(c_i-i)\bmod 26.
$$

索引必须来自原始整段字符串，不能因为遇到空格或标点而暂停计数：

```python
from string import ascii_lowercase, ascii_uppercase

ciphertext = "Ypw'zj zwufpp hwu txadjkcq dtbtyu kqkwxrbvu! Mbz cjzg kv IAJBO{ndldie_al_aqk_jjrnsxee}. Xzi utj gnn olkd qgq ftk ykaqe uei mbz ocrt qi ynlu, etrm mff'n wij bf wlny mjcj :)."

plaintext = []
for i, char in enumerate(ciphertext):
    if char in ascii_lowercase:
        alphabet = ascii_lowercase
    elif char in ascii_uppercase:
        alphabet = ascii_uppercase
    else:
        plaintext.append(char)
        continue
    plaintext.append(alphabet[(alphabet.index(char) - i) % 26])

print(''.join(plaintext))
```

输出中的 flag 为：

```text
DUCTF{crypto_is_fun_kjqlptzy}
```

## 方法总结

题名中的 `i` 是本题最关键的变化量：位移量随绝对字符索引增长。处理这类变种 Caesar 密码时，应先用已知格式或自然语言词形验证索引、方向和计数规则，再实现逐位置逆变换；尤其要确认标点是否参与位置计数。
