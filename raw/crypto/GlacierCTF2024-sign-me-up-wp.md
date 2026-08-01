# GlacierCTF 2024 Sign me up

## 题目简述

服务实现了一个修改版 Ed25519 签名 oracle：选手可以请求任意消息的签名，结束查询后必须为服务器随机生成的 32 字符串伪造签名。官方 WP 只有 TODO，但 Rust 源码和官方 Sage exploit 完整给出了预期漏洞。

实现把 Ed25519 原本使用的 SHA-512 换成 SHA-1，并把 20 字节摘要放入 64 字节小端缓冲区后转成标量。结果是每次签名的 nonce $r_i$ 只有 160 位，而群阶约为 252 位；多组签名构成 Hidden Number Problem，可用格上的 CVP 恢复私钥标量。

## 解题过程

### 1. 建立弱 nonce 的签名方程

代码计算：

```rust
hash_val[0..20].copy_from_slice(Sha1(prefix || message));
let r = Scalar::from_bytes_mod_order_wide(&hash_val);
let R = (r * G).compress();

hash_val[0..20].copy_from_slice(Sha1(R || public_key || message));
let h = Scalar::from_bytes_mod_order_wide(&hash_val);
let s = h * secret_scalar + r;
```

由于高 44 字节全为 0，并按小端整数解释，$0\le r_i<2^{160}$，挑战哈希 $h_i$ 也小于 $2^{160}$。每组签名满足：

$$
s_i\equiv r_i+h_i a\pmod\ell,
$$

其中 $a$ 是私钥标量，$\ell$ 是 Ed25519 基点阶：

```text
0x1000000000000000000000000000000014def9dea2f79cd65812631a5cf5d3ed
```

### 2. 用 CVP 恢复私钥标量

官方 exploit 请求消息 `0` 到 `4` 的 5 组签名，并构造 6 维有理格。第一行编码 $a/2^{100}$ 和 $h_i a$，其余对角线放置群阶 $\ell$：

```text
[ 2^-100   h1   h2   h3   h4   h5 ]
[ 0        l    0    0    0    0  ]
[ 0        0    l    0    0    0  ]
[ 0        0    0    l    0    0  ]
[ 0        0    0    0    l    0  ]
[ 0        0    0    0    0    l  ]
```

由 $h_i a\equiv s_i-r_i\pmod\ell$，各签名对应坐标距离 $s_i-2^{159}$ 至多约 $2^{159}$。目标向量取：

```text
[ l/(2*2^100), s1-2^159, ..., s5-2^159 ]
```

对格做 LLL 约化，再用 Babai nearest-plane 求近似最近向量。结果第一坐标乘回 $2^{100}$，即可得到私钥标量 $a$。恢复后应把它代回全部签名方程，确认导出的 $r_i=(s_i-h_i a)\bmod\ell$ 均落在 $[0,2^{160})$，避免接受错误近似。

### 3. 复用旧 $R$ 伪造随机消息

不必知道旧 nonce 的离散对数。取第 1 个签名的 $(R,s_0)$，分别计算旧消息和新挑战的哈希 $h_0,h_*$。由：

$$
s_0=r_0+h_0a
$$

直接构造：

$$
s_*=s_0+(h_*-h_0)a\pmod\ell.
$$

这样 $s_*=r_0+h_*a$，与同一个 $R=r_0G$ 完全匹配。提交 `R.hex()` 和 `s_*.hex()` 后通过验证，得到：

```text
gctf{L3N57ra_L3n57RA_aND_LoVa5Z_7O_7h3_r35CU3}
```

## 方法总结

本题展示了签名哈希长度与标量域不匹配造成的格攻击。哈希碰撞并不是主线；真正泄漏来自 nonce 只覆盖 160 位区间，使 $s_i-h_i a$ 成为已知的小误差。Ed25519 应严格使用标准化算法和 SHA-512 派生 nonce，不应自行替换哈希或截断摘要；恢复私钥后，复用任一旧 $R$ 就能代数修正 $s$，为任意新消息伪造签名。
