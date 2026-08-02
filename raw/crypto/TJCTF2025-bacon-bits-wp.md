# bacon-bits

## 题目简述

附件将 flag 中的每个英文字母映射为 5 位 Bacon 密码，再用载体文本中连续 5 个字符的大小写表示 0 和 1。生成器最后还把每个载体字符的码点减去 13，因此密文表面既看不出原始文字，也看不出大小写编码。题目规定 flag 全部小写且格式为 `tjctf{...}`，这一条件用于消除 Bacon 码中 `i/j` 与 `u/v` 共用码字造成的歧义。

## 解题过程

生成器的两层处理必须逆序撤销：先给每个密文字符的码点加 13，再按 5 个字符一组把小写记为 `0`、大写记为 `1`。最后用附件中的 Bacon 表反查字母。

```python
TABLE = {
    "00000": "a", "00001": "b", "00010": "c", "00011": "d",
    "00100": "e", "00101": "f", "00110": "g", "00111": "h",
    "01000": "i", "01001": "k", "01010": "l", "01011": "m",
    "01100": "n", "01101": "o", "01110": "p", "01111": "q",
    "10000": "r", "10001": "s", "10010": "t", "10011": "u",
    "10100": "w", "10101": "x", "10110": "y", "10111": "z",
}

with open("out.txt", "r", encoding="utf-8") as f:
    carrier = "".join(chr(ord(ch) + 13) for ch in f.read().strip())

decoded = []
for start in range(0, len(carrier), 5):
    group = carrier[start:start + 5]
    bits = "".join("0" if ch.islower() else "1" for ch in group)
    decoded.append(TABLE[bits])

print("".join(decoded))
```

直接反查得到的连续字母是 `tictfoinkooinkoooinkooooink`。这不是脚本故障：附件明确令 `i` 与 `j` 同为 `01000`，令 `u` 与 `v` 同为 `10011`，单靠 Bacon 码无法区分。结合题目给定的 `tjctf{...}` 格式，把前缀的第二个字母恢复为 `j`，并把非字母的花括号补回，最终得到：

```text
tjctf{oinkooinkoooinkooooink}
```

## 方法总结

- 核心技巧：码点平移还原载体后，通过字符大小写读取 Bacon 二进制码。
- 识别信号：每 5 个字符一组、内容本身无意义但大小写分布异常，通常应检查 Bacon 密码。
- 复用要点：经典 24 字母 Bacon 表会合并 `i/j` 和 `u/v`；输出存在歧义时只能依靠已知格式或语义约束消歧，不能声称编码本身给出了唯一答案。
