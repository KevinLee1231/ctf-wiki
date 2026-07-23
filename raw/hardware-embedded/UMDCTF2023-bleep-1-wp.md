# Bleep 1

## 题目简述

题目给出 Logisim Evolution 电路 `Bleep1.circ` 和十六进制数据 `flag.enc`，提示把数据装入 ROM 后运行。电路通过计数器逐字节读取 ROM，将八条位线重新排列，再送入 TTY。

生成器表明每个原始字节的位串
$b_0b_1\ldots b_7$ 按从最高位到最低位编号，并被重排为：

$$
e=b_2b_3b_1b_5b_4b_6b_7b_0.
$$

## 解题过程

最直接的方法是在 Logisim Evolution 3.8 中打开电路，右击 ROM 加载
`flag.enc`，复位 TTY 后启动时钟。计数器逐地址读取 ROM，分线器完成逆置换，TTY 会直接输出 flag。

也可以不运行图形界面，按电路连线逆向位排列。由上述关系得到：

$$
b=b_0b_1b_2b_3b_4b_5b_6b_7
=e_7e_2e_0e_1e_4e_3e_5e_6.
$$

逐字节解码：

```python
encoded = bytes.fromhex(open("flag.enc").read().strip())
decoded = bytearray()

for value in encoded:
    e = f"{value:08b}"
    b = "".join(e[index] for index in [7, 2, 0, 1, 4, 3, 5, 6])
    decoded.append(int(b, 2))

print(decoded.decode())
```

位序中的索引 0 是格式化二进制串的最高位，不能按最低有效位重新编号。结果为：

```text
UMDCTF{w3lc0me_t0_l0g1s1m_yeet}
```

## 方法总结

- 核心技巧：沿 Logisim 的 ROM、分线器和 TTY 数据路径恢复固定八位置换，再对每个密文字节应用逆置换。
- 识别信号：数字电路中没有算术或状态混淆，只有八条位线交叉连接时，本质通常是 bit permutation。
- 复用要点：明确图形分线器的 bit 0 对应最低位还是字符串左端；先用 `U` 的 ASCII 值验证位序，再批量解码其余字节。
