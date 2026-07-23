# haes_2

## 题目简述

HAES 2 沿用 HAES 的 64 字节密钥和 $4\times4\times4$ 状态，但把轮数从 5 降到 4。服务仍先返回加密 flag 和选择明文加密接口，不过输入必须是可打印文本，而且最多只能提交三个 64 字节分组，无法再发送完整的 256 元积分集合。

四轮结构仍然过短。只需提交三个在同一字节上有差异的明文，就能跟踪差分经过线性层后的活动位置，结合 AES S 盒的差分分布表恢复第四轮轮密钥。

## 解题过程

### 获取三组选择明文

官方脚本构造三个 64 字节分组，只有第一个字节分别为 `A`、`B`、`C`，其余字节均为 `A`。它们满足服务的可打印字符限制，并提供三对可交叉验证的差分：

```python
pts = [
    (chr(ord("A") + i) + "A" * 63).encode()
    for i in range(3)
]
query = b"".join(pts)
```

把三个返回分组记为 $C_0,C_1,C_2$。末轮只有
`SubBytes -> ShiftPlanes -> AddRoundKey`，所以先对密文差分执行
`inv_shift_planes`，就能把观察值对齐到末轮 S 盒输出位置。

### 构造 S 盒差分分布表

对所有 $x,y\in\mathbb{F}_{2}^{8}$ 预计算

$$
\Delta_{\text{in}}=x\oplus y,\qquad
\Delta_{\text{out}}=S(x)\oplus S(y),
$$

并按 $(\Delta_{\text{in}},\Delta_{\text{out}})$ 保存可能的输入对：

```python
ddt = [[[] for _ in range(256)] for _ in range(256)]
for x in range(256):
    for y in range(256):
        ddt[x ^ y][s_box[x] ^ s_box[y]].append((x, y))
```

题目线性层完全公开。把只有首字节活动的符号状态依次通过
`MixColumns` 和 `ShiftPlanes`，即可确定进入末轮前每个四字节列中哪个字节最初活动。对该活动字节枚举 256 种差分值，再经过一次
`mix_single_column`，得到末轮 S 盒四个输入字节可能的差分。

### 按列恢复第四轮密钥

对每个四字节列和每一对密文：

1. 撤销末轮 `ShiftPlanes`，取得观测到的四字节 S 盒输出差分。
2. 枚举该列可能的 S 盒输入差分。
3. 从 DDT 查出每个字节可能的 S 盒输入对 $(x,y)$。
4. 由 $C=S(x)\oplus K_4$ 计算候选轮密钥字节
   $K_4=S(x)\oplus C$。
5. 将三对密文分别产生的候选集合取交集。

核心关系可写成：

```python
for sbox_in_0, sbox_in_1 in ddt[input_diff][output_diff]:
    key_byte = s_box[sbox_in_0] ^ ct_0_byte
    assert key_byte == s_box[sbox_in_1] ^ ct_1_byte
    byte_candidates.append(key_byte)
```

四字节候选做笛卡尔积后按列求交，通常每列只剩一个结果。处理 16 列并恢复正确的 `ShiftPlanes` 排列后，即得到完整的 64 字节第四轮密钥。

### 恢复主密钥和 flag

密钥扩展可逆，因此从第四轮密钥逆推四次即可得到主密钥，再按题目给出的逆变换解密服务最初返回的密文：

```python
master_key = bytes(inv_expand_key(round_key_4, 4))
flag = decrypt(master_key, encrypted_flag).rstrip(b"\x00")
print(flag)
```

最终结果为：

```text
UMDCTF{differential_cryptanalysis_isnt_so_bad_right?}
```

## 方法总结

- 核心技巧：用三个选择明文建立差分，依据公开线性层传播活动字节，再用 S 盒 DDT 反推出末轮密钥。
- 识别信号：低轮 SPN、末轮省略线性混合、选择明文数量很少但允许精确控制差分时，应考虑按列或按字节的差分密钥恢复。
- 复用要点：不要直接暴力猜完整轮密钥；先利用线性层把问题拆成独立四字节列，再用多组密文对的候选交集消除 DDT 的多解。
