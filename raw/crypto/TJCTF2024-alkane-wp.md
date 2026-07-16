# alkane

## 题目简述

题目给出一个 16 字节明文 `yellow submarine`、对应密文，以及生成密钥流的 `main.py`。`main.py` 将 `flag.txt` 中的 `tjctf{...}` 去掉前后缀后作为 16 字节 key，再用 `schedule` 表从 key 的 128 个 bit 中线性组合出 128 bit 密钥流，最后与明文异或得到密文。

关键机制是：每个密钥流 bit 都是 key 若干 bit 在 GF(2) 上的 XOR。因此已知一组明密文就等价于得到一个 128 元 GF(2) 线性方程组，求出 key 后再包回 `tjctf{}` 即为 flag。

## 解题过程

### 关键观察

加密函数本质是：

```python
keystream = keysch(key)
ciphertext = plaintext ^ keystream
```

而 `keysch` 的第 `i` 个输出 bit 由 `schedule[i]` 指定的若干 key bit XOR 得到。先用 `keystream = plaintext ^ ciphertext` 得到方程右端，再按 `schedule` 构造 128 行增广矩阵即可。

### 求解步骤

将每行矩阵的 `schedule[i]` 位置置 1，最后一列放入对应 keystream bit，在 GF(2) 上做高斯消元。消元后只剩一个自由变量，因此枚举两个候选 key，筛出可打印 ASCII 的那个。

核心脚本如下：

```python
d = open("main.py", "r", encoding="utf-8").read()
schedule = eval(d[:d.find("key = open")].strip()[len("schedule = "):])

message = b"yellow submarine"
ciphertext = bytes([
    0xb7, 0x8e, 0xb3, 0xd9, 0xfd, 0xf2, 0x1f, 0xa2,
    0xea, 0x7a, 0xe3, 0x0f, 0x00, 0x78, 0x6a, 0x08,
])

keystream = bytes(a ^ b for a, b in zip(message, ciphertext))
target = [(keystream[i // 8] >> (7 - i % 8)) & 1 for i in range(128)]

mat = []
for i in range(128):
    row = [0] * 129
    for bit in schedule[i]:
        row[bit] ^= 1
    row[-1] = target[i]
    mat.append(row)

pivot_row = [-1] * 128
pivots = []
col = 0
for row in range(128):
    while col < 128 and all(mat[r][col] == 0 for r in range(row, 128)):
        col += 1
    if col == 128:
        break
    found = next(r for r in range(row, 128) if mat[r][col])
    mat[row], mat[found] = mat[found], mat[row]
    pivot_row[col] = row
    pivots.append(col)
    for r in range(128):
        if r != row and mat[r][col]:
            mat[r] = [a ^ b for a, b in zip(mat[r], mat[row])]
    col += 1

free = [i for i in range(128) if pivot_row[i] == -1]
particular = [0] * 128
for c in pivots:
    particular[c] = mat[pivot_row[c]][-1]

for mask in range(1 << len(free)):
    sol = particular[:]
    for j, fv in enumerate(free):
        if mask >> j & 1:
            sol[fv] ^= 1
            for c in pivots:
                sol[c] ^= mat[pivot_row[c]][fv]

    key = bytearray(16)
    for i, bit in enumerate(sol):
        key[i // 8] |= bit << (7 - i % 8)
    if all(32 <= b <= 126 for b in key):
        print(f"tjctf{{{key.decode()}}}")
```

运行结果：

```text
tjctf{l1neeruwu8215413}
```

## 方法总结

- 核心技巧：把自定义 bit 级 key schedule 识别为 GF(2) 线性系统。
- 识别信号：已知明文、异或密钥流、每个输出 bit 由若干输入 bit XOR 得到时，应优先考虑线性代数而不是爆破。
- 复用要点：bit 顺序要和题目一致；自由变量较少时可以枚举候选，并用可打印字符或正向加密校验筛选。
