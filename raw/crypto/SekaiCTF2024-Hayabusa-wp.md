# はやぶさ (Hayabusa)

## 题目简述

服务每次生成一组低维 Falcon 密钥，公开维数 $n=64$ 的公钥多项式 $h$，要求为固定消息 `Can you break me` 提交一份有效签名。

Falcon 验证本质上检查短向量关系：

$$
s_0+h s_1=c \pmod{x^n+1,\ q},
$$

其中 $q=12289$，$c$ 由消息和 $40$ 字节 salt 经 SHAKE256 映射得到。签名只需编码 salt 和 $s_1$；验证端据此重建 $s_0=c-hs_1$，再检查 $\lVert s_0\rVert^2+\lVert s_1\rVert^2$ 是否低于参数上界。

## 解题过程

### 构造 NTRU/SIS 格

在商环 $R_q=\mathbb{Z}_q[x]/(x^n+1)$ 中，把“乘以 $h$”写成 $n\times n$ 的整数矩阵 $H$。所有满足模关系的向量可以由下列 $2n$ 维格表示：

$$
\begin{pmatrix}
qI & 0\\
-H & I
\end{pmatrix}.
$$

为了寻找靠近目标 $(c,0)$ 的短向量，再加入 Kannan embedding 列和目标行。官方求解器使用 $K=\lfloor\sqrt q\rfloor$：

```python
K = int(sqrt(q))
H = Matrix(ZZ, [(h * x^i).list() for i in range(n)])

L = block_matrix([
    [q * identity_matrix(ZZ, n), zero_matrix(n, n)],
    [-H,                         identity_matrix(ZZ, n)],
], subdivide=False)

L = L.augment(vector([0] * (2 * n)))
L = L.stack(vector(hashed + [0] * n + [K]))
reduced = L.LLL()
```

维数只有 $64$，题目参数远低于正常 Falcon 安全级别，LLL 可以找到足够短的伪造向量。

### 取出签名向量

在约化基中寻找最后一维等于 $K$ 的行。该行前 $n$ 项是 $s_0$，中间 $n$ 项是 $s_1$：

```python
for row in reduced:
    if row[-1] == K:
        s0 = R(row[:n])
        s1 = R(row[n:2*n])
        break

assert R(hashed) == s0 + h * s1
```

这里的 salt 可以自行随机选择，只需先用相同 salt 对固定消息执行 `hash_to_point`。

### 编码并提交

Falcon 压缩格式对每个 $s_1$ 系数编码符号位、低 $7$ 位以及高位的一元码。参数头为：

```python
header = bytes([0x30 + logn[64]])  # 0x36
signature = header + salt + compress(
    s1.list(),
    Params[64]["sig_bytelen"] - 1 - 40
)
```

把签名转成十六进制发送。验证器重建出的 $(s_0,s_1)$ 满足同余关系和范数界，因此返回 flag。

仓库中服务端使用的 flag 为：

```text
SEKAI{1_l1k3_7h15_mu51c_h0w_4b0u7_y0u_:https://www.youtube.com/watch?v=uUvthLpSHrQ}
```

这里的 YouTube URL 是 flag 字符串本身的一部分，不是 WP 需要依赖的外部资料，因此必须原样保留。

## 方法总结

- 核心技巧：把 Falcon 验证方程转成 NTRU/SIS 格，并用 Kannan embedding 寻找目标陪集中的短向量。
- 识别信号：Falcon 参数维数异常小，服务允许攻击者自由选择 salt，且只要求为固定消息生成一份签名。
- 复用要点：伪造后仍需严格遵循签名序列化格式；数学上找到短向量并不等于验证器会接受，header、salt 长度、系数符号和一元压缩都必须与实现一致。
