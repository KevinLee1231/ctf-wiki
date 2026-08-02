# TSGCTF2024 Who is the Outlier?

## 题目简述

系统使用模数：

$$p=2^{128}$$

秘密密钥 $s$ 是长度 $n=1024$ 的二进制向量。共有 $m=2048$ 份投票密文，令 $q=4096$、$\Delta=p/q$。除一份异常票外，所有密文都满足：

$$b_i=\langle s,a_i\rangle+\Delta\pmod p$$

异常票则满足：

$$b_j=\langle s,a_j\rangle\pmod p$$

附件还给出逐字符加密的 flag。目标是从大量方程中识别唯一异常样本、恢复二进制密钥并解密。

## 解题过程

### 1. 把正常投票改写成线性方程

对每行定义：

$$A_i=-a_i,\qquad b'_i=\Delta-b_i\pmod p$$

正常行变为：

$$\langle A_i,s\rangle=b'_i\pmod p$$

异常行的右侧多出一个 $\Delta$：

$$b'_j-\langle A_j,s\rangle=\Delta$$

所以问题转化为在 $\mathbb Z/2^{128}\mathbb Z$ 上求解一个含一行错误的超定线性系统，并要求所有未知量只能取 0 或 1。

### 2. 在 $2$ 的幂模环上消元

$\mathbb Z/2^{128}\mathbb Z$ 不是域，偶数没有乘法逆元；只有奇数是单位。消元时必须为当前列寻找奇数 pivot。若当前行系数为偶数而下方某行是奇数，可以把两行相加，使 pivot 变为奇数，再求逆：

```python
if A[row][col] % 2 == 0:
    for row2 in range(row + 1, rows):
        if A[row2][col] % 2 == 1:
            A[row] = [(x + y) % p for x, y in zip(A[row], A[row2])]
            b[row] = (b[row] + b[row2]) % p
            break

inv = pow(A[row][col], -1, p)
A[row] = [x * inv % p for x in A[row]]
b[row] = b[row] * inv % p
```

随后消去其他行该列。若从当前行向下整列都只有偶数，则把该列记为 `skipped_idx`，不强行对非单位求逆。

### 3. 绕开唯一异常行并恢复二进制解

把 2048 行分成前后两组，每组 1024 行。唯一异常行只会落在其中一组，因此至少有一组是完全一致的方程组。对两组分别消元。

消元后枚举所有被跳过列的二进制取值；题目实例中的跳过列很少，因此 $2^k$ 枚举可行。先用底部剩余方程过滤，再回代各 pivot 变量，并要求结果仍为 0 或 1：

```python
for assignment in product([0, 1], repeat=len(skipped_idx)):
    candidate = [0] * n
    for col, bit in zip(skipped_idx, assignment):
        candidate[col] = bit

    # 检查只含 skipped 变量的剩余行，再回代 pivot。
    # 若某个 pivot 解不在 {0, 1}，立即丢弃。
```

最后用全部 2048 行验证候选。正确密钥必须产生 2047 个零残差和一个 $\Delta$ 残差：

```python
residuals = [
    (b2[i] - dot(A[i], secret, p)) % p
    for i in range(m)
]
assert residuals.count(0) == m - 1
assert residuals.count(p // q) == 1
```

### 4. 解密 flag

flag 每个字符的密文满足：

$$b=\langle s,a\rangle+\operatorname{ord}(c)\Delta+e\pmod p$$

其中噪声 $0\le e<\Delta$。恢复 $s$ 后：

$$
\operatorname{ord}(c)
=\left\lfloor
\frac{(b-\langle s,a\rangle)\bmod p}{\Delta}
\right\rfloor
$$

```python
chars = []
for row in encrypted_flag:
    scaled = (row[n] - dot(row[:n], secret, p)) % p
    chars.append(chr(scaled // (p // q)))
print(''.join(chars))
```

得到：

```text
TSGCTF{f0llow_+|-|e_(om|*4s$_0f_your_|-|3art_27d6374afee804ce}
```

## 方法总结

本题的关键是同时利用“模数是 $2$ 的幂”和“密钥分量是二进制”两项结构。环上高斯消元不能随意除法，只能选择奇数单位作 pivot；无法消去的少量列则用二进制枚举补齐。把数据分半可确保至少一个子系统不含异常票，最后再以全量残差分布识别正确候选。恢复密钥后，flag 加密中的噪声严格小于缩放因子 $\Delta$，整数除法即可还原字符。
