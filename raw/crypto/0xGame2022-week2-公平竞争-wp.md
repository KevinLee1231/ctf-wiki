# week2公平竞争

## 题目简述

附件给出一个 $6\times6$ 字符方阵和密文 `3umgkggklyjs47xg0mx3`。方阵按行排列 `0-9a-z`，题目提示明文格式为 `0xGame{...}`。这不是标准的 $5\times5$ Playfair，而是为容纳数字和 26 个字母扩展出的 36 字符 Playfair。

## 解题过程

把密文按两个字符一组拆分，并按 Playfair 的逆规则处理：

- 同行：两个字符都向左移动一格，越界时循环；
- 同列：两个字符都向上移动一格，越界时循环；
- 不同行且不同列：保持各自行号，交换列号。

```python
alphabet = "0123456789abcdefghijklmnopqrstuvwxyz"
size = 6
pos = {ch: divmod(i, size) for i, ch in enumerate(alphabet)}


def decrypt_pair(a, b):
    ra, ca = pos[a]
    rb, cb = pos[b]

    if ra == rb:
        return (
            alphabet[ra * size + (ca - 1) % size]
            + alphabet[rb * size + (cb - 1) % size]
        )
    if ca == cb:
        return (
            alphabet[((ra - 1) % size) * size + ca]
            + alphabet[((rb - 1) % size) * size + cb]
        )
    return alphabet[ra * size + cb] + alphabet[rb * size + ca]


cipher = "3umgkggklyjs47xg0mx3"
plain = "".join(
    decrypt_pair(cipher[i], cipher[i + 1])
    for i in range(0, len(cipher), 2)
)
print(plain)
```

得到带填充的明文：

```text
0xgameemmxmp1ayf4irx
```

其分组是 `0x ga me em mx mp 1a yf 4i rx`。Playfair 在一对字符相同时会插入 `x`，明文长度为奇数时也会在末尾补 `x`。因此 `mx mp` 中间的 `x` 是为拆开连续的 `mm` 而插入，末尾 `rx` 中的 `x` 是补位。删除这两个填充字符，再按题目给出的 flag 格式补回大写和花括号，得到：

```text
0xGame{emmmp1ayf4ir}
```

## 方法总结

识别 Playfair 后还要确认字符表尺寸；本题出现数字，必须使用附件给出的 $6\times6$ 方阵，不能套用合并 `i/j` 的传统 $5\times5$ 字母表。解密结果中的 `x` 也不能全部机械删除，只能结合二元分组、重复字符规则、末尾补位和已知 flag 格式判断哪些是填充。
