# L3akCTF 2025 Magical Oracle Writeup

## 题目简述

服务端生成一个 256 位素数 $p$ 和秘密整数 $\alpha\in[1,p-1]$。每次查询会随机选择 $t_i$，计算

$$
x_i=\alpha t_i\bmod p,
$$

再返回一个与 $x_i$ 很接近的整数 $z_i$。最多可以查询 35 次。flag 使用

$$
K=\operatorname{SHA256}(\operatorname{str}(\alpha))
$$

作为 AES-CBC 密钥加密。

这正是 Hidden Number Problem：已知若干个乘积模 $p$ 的近似值，要求恢复公共乘数 $\alpha$。

## 解题过程

### 写出小误差关系

令返回误差为

$$
e_i=\alpha t_i-z_i-k_ip,
$$

则

$$
\alpha t_i-z_i\equiv e_i\pmod p.
$$

题目参数为：

$$
n=\operatorname{bitlength}(p)=256,
$$

$$
k=\lfloor\sqrt n\rfloor+\operatorname{bitlength}(n)+1=26,
$$

$$
d=2\lfloor\sqrt n\rfloor+3=35.
$$

服务端要求 $|x_i-z_i|<p/2^{k+1}$；即使随机候选没有命中，回退分支也只加入同数量级的小噪声。因此 35 组 $(t_i,z_i)$ 都是高质量的 HNP 样本。

### 把 HNP 改写成 CVP

构造维数为 $d+1$ 的格，基向量为

$$
B=
\begin{pmatrix}
p&0&\cdots&0&0\\
0&p&\cdots&0&0\\
\vdots&&\ddots&&\vdots\\
0&0&\cdots&p&0\\
t_1&t_2&\cdots&t_d&1/p
\end{pmatrix}.
$$

对整数系数 $(k_1,\ldots,k_d,\alpha)$，格向量为

$$
(k_1p+\alpha t_1,\ldots,k_dp+\alpha t_d,\alpha/p).
$$

它的前 $d$ 个坐标与目标向量

$$
\mathbf{z}=(z_1,\ldots,z_d,0)
$$

之差正是小误差 $(e_1,\ldots,e_d)$，最后一维则把 $\alpha$ 缩放到与误差相近的量级。因此，离 $\mathbf z$ 最近的格点最后一个坐标乘以 $p$，就是候选 $\alpha$。

官方脚本实际使用带 $1/p$ 的有理基，而不是只在说明中出现的整数近似版本：

```python
B = Matrix(QQ, d + 1, d + 1)
for i in range(d):
    B[i, i] = p
for i in range(d):
    B[d, i] = ts[i]
B[d, d] = 1 / p

target = vector(QQ, zs + [0])
B = B.LLL()
closest = babai_closest_vector(B, target)
alpha = ZZ(closest[-1] * p) % p
```

Babai 最近平面算法从最后一个 Gram-Schmidt 方向开始，把目标与基向量的投影系数取整，逐步得到一个足够接近的格点：

```python
def babai_closest_vector(B, target):
    G = B.gram_schmidt()[0]
    diff = target
    for i in reversed(range(B.nrows())):
        c = ((diff * G[i]) / (G[i] * G[i])).round()
        diff -= c * B[i]
    return target - diff
```

### 解密 flag

取得候选后，应先检查所有样本的模距离确实很小，防止 Babai 返回错误邻点：

```python
def centered(x, p):
    x %= p
    return min(x, p - x)

assert all(centered(alpha * t - z, p) < (p >> 26)
           for t, z in zip(ts, zs))
```

然后按服务端完全相同的字符串编码方式派生密钥：

```python
key = hashlib.sha256(str(alpha).encode()).digest()
raw = base64.b64decode(encrypted_flag)
iv, ct = raw[:16], raw[16:]
flag = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
print(flag)
```

最终得到：

```text
L3AK{hnp_BBB_cvp_4_the_w1n}
```

## 方法总结

看到“$\alpha t\bmod p$ 的近似值”就应联想到 Hidden Number Problem。将未知的模数倍数和 $\alpha$ 一起作为格系数，可以把恢复秘密转成 CVP；LLL 先改善基，Babai 再寻找目标附近的格点。实现时最容易出错的是缩放：最后一维的 $1/p$ 用来平衡 256 位秘密与小误差，恢复后必须再乘 $p$，并用全部样本验证候选。
