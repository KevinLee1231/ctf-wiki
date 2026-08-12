# DownUnderCTF 2022 time locked Writeup

## 题目简述

题目用七阶线性递推在有限域 $\operatorname{GF}(p)$ 上计算 $f(n)$，再以 `SHA256(str(f(n)))` 作为 AES-ECB 密钥。真正的障碍是索引

$$
n=2^{2^{1337}},
$$

直接循环不可能完成。递推系数和七个初值均公开，因此应把状态转移写成矩阵幂，并利用有限群中的周期约简指数。

## 解题过程

把状态写成 $(f_i,f_{i-1},\ldots,f_{i-6})^T$，转移矩阵为：

$$
M=
\begin{pmatrix}
a_0&a_1&\cdots&a_6\\
1&0&\cdots&0\\
&\ddots&\ddots&\vdots\\
0&\cdots&1&0
\end{pmatrix}.
$$

该矩阵属于 $GL(7,p)$，其阶整除有限群大小：

$$
|GL(7,p)|=\prod_{k=0}^{6}(p^7-p^k).
$$

因此先在该模数下计算巨大指数，再做一次快速矩阵幂。原函数从初始向量 $(\alpha_6,\ldots,\alpha_0)$ 出发，需要推进 $n-6$ 次：

```python
M = Matrix(GF(p), [a])
M = M.stack(
    Matrix.identity(6).augment(Matrix.column([0] * 6))
)

group_order = prod(p^7 - p^k for k in range(7))
exponent = int(pow(2, 2^1337, group_order)) - 6
fn = (M^exponent * vector(alpha[::-1]))[0]
```

最后派生 AES 密钥并解密：

```python
key = sha256(str(fn).encode()).digest()
flag = unpad(AES.new(key, AES.MODE_ECB).decrypt(ciphertext), 16)
```

得到：

```text
DUCTF{p4y_t0_w1n_91ea0a7b4b688fc8}
```

## 方法总结

固定阶数的线性递推应优先转成伴随矩阵，而不是按索引循环。状态位于有限域时，矩阵幂序列必然周期化；若转移矩阵可逆，可以用 $GL(d,p)$ 的群阶作为安全的指数约简模数。需要同时核对状态向量顺序和初始索引偏移，本题的 `-6` 正是最容易写错的边界。
