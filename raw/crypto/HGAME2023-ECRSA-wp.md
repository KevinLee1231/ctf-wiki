# ECRSA

## 题目简述

题目把 RSA 的“公钥乘法 / 私钥逆运算”搬到椭圆曲线上。模数为 $n=pq$，明文 flag 的整数作为点 $P$ 的横坐标，密文点为 $C=eP$，附件只给出 $C$ 的横坐标。由于 $p$、$q$ 也被公开，需要分别在 $\mathbb F_p$ 与 $\mathbb F_q$ 上恢复并解密点，再用中国剩余定理合并明文横坐标。

## 解题过程

加密关系为：

$$
C=eP.
$$

在素域曲线 $E(\mathbb F_p)$ 上，若

$$
d_p\equiv e^{-1}\pmod{\#E(\mathbb F_p)},
$$

则 $d_pC=P$；模 $q$ 同理。题目只输出密文点的横坐标 $x_C$，先用曲线方程

$$
y_C^2=x_C^3+ax_C+b
$$

补出一个合法纵坐标。`lift_x` 可能返回 $C$ 或 $-C$，但二者解密后为 $P$ 或 $-P$，横坐标相同，所以不会影响最终 flag。

完整公开参数与 Sage 解密脚本如下：

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes
from sage.all import EllipticCurve, GF, Integer, crt, inverse_mod

p = 115192265954802311941399019598810724669437369433680905425676691661793518967453
q = 109900879774346908739236130854229171067533592200824652124389936543716603840487
a = 34573016245861396068378040882622992245754693028152290874131112955018884485688
b = 103282137133820948206682036569671566996381438254897510344289164039717355513886
e = 11415307674045871669

ciphertext = (
    b"f\xb1\xae\x08`\xe8\xeb\x14\x8a\x87\xd6\x18\x82\xaf1q"
    b"\xe4\x84\xf0\x87\xde\xedF\x99\xe0\xf7\xdcH\x9ai\x04["
    b"\x8b\xbbHR\xd6\xa0\xa2B\x0e\xd4\xdbr\xcc\xad\x1e\xa6"
    b"\xba\xad\xe9L\xde\x94\xa4\xffKP\xcc\x00\x907\xf3\xea"
)
cipher_x = bytes_to_long(ciphertext)

curve_p = EllipticCurve(GF(p), [a, b])
curve_q = EllipticCurve(GF(q), [a, b])

cipher_p = curve_p.lift_x(Integer(cipher_x))
cipher_q = curve_q.lift_x(Integer(cipher_x))

d_p = inverse_mod(e, curve_p.order())
d_q = inverse_mod(e, curve_q.order())

plain_p = d_p * cipher_p
plain_q = d_q * cipher_q

message_x = crt([Integer(plain_p[0]), Integer(plain_q[0])], [p, q])
print(long_to_bytes(int(message_x)))
```

输出为：

```text
b'hgame{ECC_4nd_RSA_also_can_be_combined}'
```

官方题解还给出了直接在 $E(\mathbb Z_n)$ 上表示密文点的路线：先分别求解模 $p$、模 $q$ 的两个平方根，并对四种符号组合做 CRT，得到密文纵坐标；再计算两条素域曲线的阶，以其最小公倍数为标量群周期求 $e$ 的逆元。它与上面的分域解密本质相同，但中间要处理四个候选点，代码更长。先在两个素域上分别解密横坐标，可以避开合数环上 `lift_x` 很慢或失败的问题。

官方 PDF 包含两种思路与参数，但未展示最终运行结果；flag 由参赛者的 [HGAME 2023 Week 4 题解](https://lazzzaro.github.io/2023/02/06/match-HGAME-2023-Week-4/index.html) 交叉核验。

## 方法总结

所谓 ECRSA 的核心仍是“在有限群中对公开指数求逆”。公开 $p,q$ 后，困难的整数分解已经消失，只需把合数模曲线拆到 $\mathbb F_p$、$\mathbb F_q$，分别按各自曲线阶求私有标量，再 CRT 合并横坐标。只给点的 $x$ 坐标时要考虑正负两个纵坐标，但椭圆曲线取负不改变横坐标，正好与本题的明文编码方式相容。
