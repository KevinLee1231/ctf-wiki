# TSGCTF2021 Flag is Win WP

## 题目简述

本题恢复了标准 secp256k1 ECDSA 公式，但随机 nonce 的生成方式存在严重结构泄露：

```ruby
k = OpenSSL::BN.rand(@curve.degree / 3).to_s.unpack1('H*').hex
x, _ = (@G * k).xy
s = (z + x * @d) * inv(k) % @n
```

`BN.rand(85).to_s` 先把约 85 位随机数转成十进制字符串，随后 `unpack1('H*').hex` 又把这些 ASCII 字节解释成整数。典型字符串长 26 位，因此 $k$ 的每个基 256 数位都只能是 `0x30` 到 `0x39`。服务允许取得三份固定消息 `Baba` 的签名；目标是利用 nonce 的窄字节范围恢复私钥，再为 `Flag` 签名。

## 解题过程

对第 $i$ 份签名 $(r_i,s_i)$，ECDSA 方程为：

$$
s_i k_i=z+r_i d\pmod n.
$$

即：

$$
k_i=\frac{r_i}{s_i}d+\frac{z}{s_i}\pmod n.
$$

这里 $z=\operatorname{SHA256}(\text{Baba})$，三份签名共享私钥 $d$。把 26 字节 nonce 写成：

$$
k_i=\sum_{j=0}^{25}(48+u_{i,j})256^j,\qquad 0\le u_{i,j}\le9.
$$

令：

$$C=48\sum_{j=0}^{25}256^j$$

减去已知的 ASCII `0` 基线后，未知量 $u_{i,j}$ 都是 0 到 9 的小整数。这样，三个签名方程可组成一个模曲线阶 $n$ 的格上最近向量问题：格基包含共享的 $d$ 系数 $r_i/s_i$、每个字节的权重 $256^j$ 以及三条模 $n$ 关系，目标向量的每组首项为 $C-z/s_i$。

官方 solver 对该格做 LLL，再用 Babai nearest plane 求最接近目标的向量：

```sage
lattice = IntegerLattice(matrix, lll_reduce=True)
reduced = lattice.reduced_basis
gram = reduced.gram_schmidt()[0]
closest = Babai_closest_vector(reduced, gram, target)
digits = [r - t for r, t in zip(closest, target)]
```

恢复第一份签名的 26 个十进制数字后，重新拼成 ASCII 字节所代表的 nonce：

```sage
k1 = int.from_bytes(
    ''.join(map(str, digits[25::-1])).encode(),
    'big',
)
```

由一份完整的 $(r_1,s_1,k_1)$ 直接求私钥：

$$
d=(s_1k_1-z)r_1^{-1}\pmod n.
$$

```sage
d = (F(k1) * F(s1) - F(z)) / F(r1)
```

得到 $d$ 后任选非零 nonce，例如官方脚本使用 $k=334$，计算 `Flag` 的合法签名：

```sage
z2 = Integer(sha256(b'Flag').hexdigest(), 16)
R = k * G
r = Integer(R[0])
s = (z2 + r * d) / k
```

把 $(r,s)$ 提交给验证接口，得到：

```text
TSGCTF{CRYPTO_IS_LOCK._KEY_IS_OPEN._CTF_IS_FUN!}
```

## 方法总结

ECDSA nonce 不仅必须保密和不重复，还必须在整个标量域中近似均匀。这里的 nonce 虽来自随机数，却先经过十进制字符串和 ASCII 编码，使每个字节只剩十种可能；三份签名就足以把私钥恢复转化为低维误差的格 CVP。修复方式是使用经过审计的 ECDSA 实现及其标准 nonce 生成过程，例如正确的系统随机采样或 RFC 6979，绝不能把格式化后的文本再当作标量。
