# ezLCG

## 题目简述

题目用 DSA 对 5 条随机消息签名，但每次签名所需的临时随机数并非独立生成，而是来自同一个未知参数的线性同余生成器（LCG）：

$$
k_{i+1}\equiv a k_i+b\pmod q.
$$

私钥 $d$ 同时被用来派生 AES-CBC 密钥：`SHA1(str(d))[:16]`。因此，核心目标是利用连续签名之间的 LCG 关系恢复 $d$，再解密密文。

## 解题过程

DSA 签名满足

$$
r_i=(g^{k_i}\bmod p)\bmod q,
$$

$$
s_i\equiv k_i^{-1}(z_i+r_i d)\pmod q,
\qquad z_i=\operatorname{SHA1}(m_i)\bmod q.
$$

这里必须保留对 $p$ 的第一次取模；不能把 $r_i$ 简写成 $g^{k_i}\bmod q$。由第二式可得

$$
k_i\equiv s_i^{-1}r_i d+s_i^{-1}z_i\pmod q.
$$

令 $A_i=s_i^{-1}r_i$、$B_i=s_i^{-1}z_i$，则每个临时随机数都是私钥的一次式：

$$
k_i=A_i d+B_i.
$$

对四个连续状态 $k_1,k_2,k_3,k_4$，LCG 的相邻差分满足

$$
k_3-k_2\equiv a(k_2-k_1),
$$

$$
k_4-k_3\equiv a(k_3-k_2)\pmod q.
$$

消去未知乘数 $a$，得到

$$
(k_4-k_3)(k_2-k_1)-(k_3-k_2)^2\equiv0\pmod q.
$$

代入 $k_i=A_i d+B_i$ 后，这是有限域 $\mathbb F_q$ 上关于 $d$ 的至多二次多项式。求出所有根，再逐一派生 AES 密钥并检查合法填充与 `moectf{` 前缀即可；不能依赖根列表的固定顺序。

```python
from hashlib import sha1

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from sage.all import GF, PolynomialRing, inverse_mod

q = 1303803697251710037027345981217373884089065173721
msg = [
    b'I\xf0\xccy\xd5~\xed\xf8A\xe4\xdf\x91+\xd4_$',
    b'~\xa0\x9bCB\xef\xc3SY4W\xf9Aa\rO',
    b'\xe6\x96\xf4\xac\n9\xa7\xc4\xef\x82S\xe9 XpJ',
    b'3,\xbb\xe2-\xcc\xa1o\xe6\x93+\xe8\xea=\x17\xd1',
    b'\x8c\x19PHN\xa8\xbc\xfc\xa20r\xe5\x0bMwJ',
]
sigs = [
    (913082810060387697659458045074628688804323008021,
     601727298768376770098471394299356176250915124698),
    (406607720394287512952923256499351875907319590223,
     946312910102100744958283218486828279657252761118),
    (1053968308548067185640057861411672512429603583019,
     1284314986796793233060997182105901455285337520635),
    (878633001726272206179866067197006713383715110096,
     1117986485818472813081237963762660460310066865326),
    (144589405182012718667990046652227725217611617110,
     1028458755419859011294952635587376476938670485840),
]
iv = b'M\xdf\x0e\x7f\xeaj\x17PE\x97\x8e\xee\xaf:\xa0\xc7'
ct = (
    b"\xa8a\xff\xf1[(\x7f\xf9\x93\xeb0J\xc43\x99\xb25:\xf5>\x1c?\xbd\x8a"
    b"\xcd)i)\xdd\x87l1\xf5L\xc5\xc5'N\x18\x8d\xa5\x9e\x84\xfe\x80\x9dm\xcc"
)

z = [int.from_bytes(sha1(m).digest(), "big") % q for m in msg]
A = [r * inverse_mod(s, q) % q for r, s in sigs]
B = [zi * inverse_mod(s, q) % q for zi, (_, s) in zip(z, sigs)]

R = PolynomialRing(GF(q), "d")
d = R.gen()
k = [A[i] * d + B[i] for i in range(5)]
f = (k[4] - k[3]) * (k[2] - k[1]) - (k[3] - k[2]) ** 2

for root, _ in f.roots():
    candidate = int(root)
    key = sha1(str(candidate).encode()).digest()[:16]
    try:
        plain = unpad(AES.new(key, AES.MODE_CBC, iv).decrypt(ct), 16)
    except ValueError:
        continue
    if plain.startswith(b"moectf{"):
        print(candidate)
        print(plain.decode())
```

运行后得到：

```text
moectf{w3ak_n0nce_is_h4rmful_to_h3alth}
```

## 方法总结

本题的决定性缺陷不是 LCG 参数本身可见，而是 DSA 临时随机数之间存在可消元的线性关系。先用签名方程把每个 $k_i$ 表成私钥 $d$ 的一次式，再通过相邻差分消去 LCG 的 $a,b$，即可把私钥恢复转化为有限域低次方程求根。实现时还应遍历并验证全部候选根，避免把求根结果的排列顺序误当成密码学依据。
