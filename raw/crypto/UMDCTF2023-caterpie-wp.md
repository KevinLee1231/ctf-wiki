# Caterpie

## 题目简述

题目实现了一个小参数 LWE 加密系统。公开密钥满足

$$
B=As+e\pmod q,
$$

其中 $A\in\mathbb Z_q^{64\times64}$，秘密向量 $s$ 和误差向量 $e$ 的分量都由中心在 0 附近的二项分布产生，模数为 $q=2^{16}$。

加密时另取小矩阵 $S'$、$E'$ 和小向量 $e'$，计算

$$
B'=S'A+E',\qquad
V=S'B+e'+\operatorname{Encode}(m)\pmod q.
$$

每个 flag 十六进制字符被编码为 $0,q/16,\ldots,15q/16$。参数维数和噪声都太小，可以用格约化恢复 $s$。

## 解题过程

### 把 LWE 写成有界距离问题

由

$$
B=As+e-cq
$$

可知，$B$ 靠近由 $A$ 和 $qI$ 生成的某个格点，偏差就是小向量
$s\Vert e$。采用行向量表示时，构造基础格：

$$
\begin{bmatrix}
I&-A^T\\
0&qI
\end{bmatrix}.
$$

这把恢复 LWE 秘密转化为有界距离解码问题。

### 使用 Kannan 嵌入

再把目标向量 $B$ 嵌入额外一维：

$$
M=
\begin{bmatrix}
I&-A^T&0\\
0&qI&0\\
0&B^T&1
\end{bmatrix}.
$$

该格包含与 $s\Vert e\Vert1$ 对应的异常短向量。由于 $n=64$ 且误差很小，对
$M$ 运行 BKZ（官方脚本使用块大小 20）即可在约化基中得到它：

```python
kannan = block_matrix([
    [identity_matrix(ZZ, n), -A.T,                  zero_matrix(n, 1)],
    [zero_matrix(n, n),      q * identity_matrix(ZZ, n), zero_matrix(n, 1)],
    [zero_matrix(1, n),       B.T,                   identity_matrix(1)],
])

reduced = kannan.BKZ(block_size=20)
s = matrix(list(reduced[0])[:n]).T
```

实际使用时应检查候选向量的符号，并验证
$B-As\bmod q$ 是否在中心提升后保持小范数，而不能盲目假定约化基第一行永远就是正确方向。

### 解密并量化消息

代入秘密向量：

$$
V-B's=S'(As+e)+e'+m-(S'A+E')s=m+S'e+e'-E's.
$$

剩余误差相对 $q/16$ 的编码间距足够小。把每个分量乘以 $16/q$、四舍五入并模 16，即可恢复十六进制字符串，再按字节解码：

```python
decoded_hex = ""
for entry in (V - B_prime * s) % q:
    nibble = round(int(entry[0]) * 16 / q) % 16
    decoded_hex += "0123456789abcdef"[nibble]

print(bytes.fromhex(decoded_hex).decode())
```

最终得到：

```text
UMDCTF{c@teRp15_U53d_BKZ!}
```

## 方法总结

- 核心技巧：将小参数 LWE 的秘密恢复写成 BDD，再用 Kannan 嵌入转成唯一短向量问题并运行 BKZ。
- 识别信号：方阵维数只有几十、模数不大、秘密和误差均为窄分布时，LWE 的理论安全性并不代表具体参数安全。
- 复用要点：格约化后要用原方程验证候选；解密还需按生成器的 nibble 间距量化，不能直接把模 $q$ 的数值当字节。
