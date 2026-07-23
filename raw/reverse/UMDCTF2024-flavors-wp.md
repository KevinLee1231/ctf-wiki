# Flavors

## 题目简述

附件是 Elixir BEAM 字节码，题目给出一串期望输出。程序要求输入恰好 47 字节，先逐字节与固定短语异或并重排，再把每 4 位经过条件置换后编码成十六进制。需要逆转两层变换恢复 flag。

## 解题过程

反编译 BEAM 或对照仓库中的 `flavors.ex`，固定密钥为：

```text
he who controls the spice controls the universe
```

函数 `a/1` 对第 $i$ 个输入字节执行：

$$
t_i=\text{input}_i\oplus\text{key}_i
$$

偶数下标把 $t_i$ 放到递归结果前，奇数下标放到递归结果后。展开递归后，实际输出顺序为：

```text
0, 2, 4, ..., 46, 45, 43, ..., 1
```

函数 `b/1` 每次读取位串中的 4 位 $(w,x,y,z)$。若 $w=y$，输出数值

$$
z+2w+4y+8x
$$

否则输出

$$
x+2y+4w+8z
$$

输入空间只有 16 种，直接枚举即可构造十六进制字符到原始 4 位的逆表。完整恢复脚本如下：

```python
desired = (
    "AD38A5970B000E1500041F0B00011617AA85109204082D1485040326051D1301"
    "2716BF081189AB990E2D0F182CA824"
)
key = b"he who controls the spice controls the universe"

def encode_nibble(bits):
    w, x, y, z = bits
    if w == y:
        value = z + 2 * w + 4 * y + 8 * x
    else:
        value = x + 2 * y + 4 * w + 8 * z
    return format(value, "X")

inverse = {}
for value in range(16):
    bits = tuple((value >> (3 - i)) & 1 for i in range(4))
    inverse[encode_nibble(bits)] = "".join(map(str, bits))

bitstream = "".join(inverse[c] for c in desired)
ordered = bytes(
    int(bitstream[i:i + 8], 2)
    for i in range(0, len(bitstream), 8)
)

indices = list(range(0, 47, 2)) + list(range(45, 0, -2))
masked = bytearray(47)
for position, original_index in enumerate(indices):
    masked[original_index] = ordered[position]

flag = bytes(masked[i] ^ key[i] for i in range(47))
print(flag.decode())
```

输出为：

```text
UMDCTF{what_about_melange_but_in_elixir_form_?}
```

## 方法总结

本题虽然载体是 BEAM，但核心只是可逆位变换。先把递归连接操作展开成明确的索引排列，再对 4 位映射穷举逆表，最后异或固定短语即可。对只有 16 个输入状态的非线性分支，枚举通常比手推布尔逆公式更稳妥。
