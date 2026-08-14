# Iron Door

## 题目简述

服务在安全素数群中工作，公开 $p=2q+1$、$g=3$ 和 $y=g^x\bmod p$。它提供签名 oracle，但拒绝任何包含 `goma` 的消息；目标是为 `hirake goma` 构造有效签名。

签名并非标准 DSA。两个哈希都截断了 SHA-256：

$$
h_1:\{0,1\}^*\rightarrow[0,2^{160}),\qquad
h_2:\{0,1\}^*\rightarrow[0,2^{200}).
$$

对实际签名消息 $m\mathbin\|salt$，服务计算：

$$
\begin{aligned}
z&=h_1(m\mathbin\|salt),\\
k&=h_2(\operatorname{I2OSP}(x+z))^{-1}\bmod q,\\
r&=h_2(\operatorname{I2OSP}(g^k\bmod p)),\\
s&=(z+xr)k^{-1}\bmod q.
\end{aligned}
$$

签名接口故意不返回 $r$，只返回 $s$。漏洞仍然存在：两次截断使隐藏乘数和误差远小于 2048 位的 $q$，可以用两层 LLL 从多份 $s$ 中恢复私钥 $x$。决定性障碍是隐藏数与格攻击，归入 `crypto`。

## 解题过程

### 把签名改写成带小量的同余式

令：

$$
\ell_i=k_i^{-1}\bmod q=h_2(\operatorname{I2OSP}(x+z_i)),
\qquad a_i=r_i\ell_i.
$$

由于 $h_2$ 的输出只有 200 位且远小于 $q$，这里的 $\ell_i$ 就是该短哈希整数本身。每份签名满足：

$$
s_i\equiv z_i\ell_i+a_i x\pmod q.
$$

各量的上界为：

$$
z_i<2^{160},\quad \ell_i<2^{200},\quad
z_i\ell_i<2^{360},\quad a_i<2^{400}.
$$

记 $B=2^{360}$。虽然 $a_i$ 不公开，但对任意两份签名，私钥项可以消去：

$$
s_j a_i-s_i a_j
\equiv
(z_j\ell_j)a_i-(z_i\ell_i)a_j
\pmod q.
$$

右侧由短哈希乘积组成，相对于 $q$ 很小。官方解法收集 20 份随机消息的 $s_i$，构造第一层格恢复 $a_i=r_i\ell_i$：

```sage
n = 20
B = 2**360

ss = [...]  # 20 个签名接口返回的 s_i
M1 = block_matrix([
    [B, matrix(ZZ, 1, n - 1, ss[1:]), 0],
    [0, matrix.identity(n - 1) * ss[0], 0],
    [0, matrix.identity(n - 1) * q, matrix.identity(n - 1)],
])

reduced1 = M1.LLL()
coeff = reduced1[0] * M1**(-1)
a = [abs(int(coeff[i])) for i in range(n)]
```

### 第二层 LLL 恢复私钥

已知 $a_i$ 后，原式变成经典隐藏数问题：

$$
s_i-a_i x\equiv z_i\ell_i\pmod q,
\qquad |z_i\ell_i|<B.
$$

官方 `solve.sage` 使用下面的有理缩放格，把小误差、私钥 $x$ 和常数 1 放入同一个短向量：

```sage
M2 = block_matrix([
    [matrix.identity(n) * q, 0, 0],
    [matrix(QQ, 1, n, a), B / q, 0],
    [matrix(QQ, 1, n, ss), 0, B],
])

reduced2 = M2.LLL()
state = reduced2[1] * M2**(-1)
x = abs(int(state[-2]))
assert pow(g, x, p) == y
```

行选择依赖 LLL 输出顺序，官方脚本也注明偶尔可能失败；公钥断言是不可省略的验证，失败时应重新连接并收集新样本，而不是继续用错误私钥。

### 为禁止消息签名

恢复 $x$ 后，客户端已经能完整复现服务端的确定性 nonce 和签名过程。对 `b"hirake goma" + salt` 计算 $(r,s)$，再把不含 salt 的原消息和签名提交到验证接口：

```python
from hashlib import sha256
from Crypto.Util.number import inverse, long_to_bytes

def h1(data):
    return int(sha256(data).hexdigest()[:40], 16)

def h2(data):
    return int(sha256(data).hexdigest()[:50], 16)

def forge(data):
    z = h1(data)
    k = inverse(h2(long_to_bytes(x + z)), q)
    r = h2(long_to_bytes(pow(g, k, p)))
    s = (z + x * r) * inverse(k, q) % q
    return r, s

target = b"hirake goma"
r, s = forge(target + salt)
```

验签通过后得到：

```text
CakeCTF{im_r3a11y_afraid_0f_truncating_hash_dig3st_13ading_unint3nd3d}
```

## 方法总结

- 核心技巧：先利用两份签名间的消元关系，从仅公开的 $s_i$ 恢复短乘积 $a_i=r_i\ell_i$；再把 $s_i-a_i x$ 视为小误差，用第二层 LLL 恢复 $x$。
- 识别信号：大素数群、自定义签名、截断哈希、确定性 nonce，以及签名分量缺失但仍存在多次查询接口。
- 复用要点：格攻击前必须写出每个隐藏量的位数上界；这里真正打开攻击面的不是“确定性”本身，而是 $h_1$、$h_2$ 截断后令 $z_i\ell_i$ 和 $r_i\ell_i$ 远小于 $q$。恢复候选私钥后始终用公开关系 $y=g^x\bmod p$ 验证。
