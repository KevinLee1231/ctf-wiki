# DownUnderCTF 2020 - impECCable

## 题目简述

服务在 NIST P-384 上实现自定义 ECDSA 风格签名。私钥为 $d$，公钥为 $Q=dG$。每次签名的 nonce 只有 340 个随机高位，低 25 位固定为私钥哈希派生出的同一值。服务最多提供 10 个短消息签名，目标是恢复 $d$，再为受保护消息 `I know alll of your secrets!` 生成合法签名。

## 解题过程

签名返回 $(r_{1i},r_{2i},s_i)$，其中：

$$
k_i=2^{25}p_i+x,qquad
s_i\equiv k_i^{-1}(h_ir_{1i}-r_{2i}d)\pmod n.
$$

$p_i$ 只有 340 位，$x$ 是所有样本共享的 25 位后缀。整理签名方程并乘以 $2^{-25}$，定义：

$$
t_i=-2^{-25}r_{2i}s_i^{-1},\qquad
a_i=2^{-25}h_ir_{1i}s_i^{-1}\pmod n.
$$

可得 $t_id+a_i=p_i+2^{-25}x$。将第一个样本与其余样本相减，公共后缀被消掉：

$$
(p_0-p_i)-(t_0-t_i)d-(a_0-a_i)\equiv0\pmod n.
$$

这就是隐藏数问题：$d$ 是隐藏数，$p_0-p_i$ 相对于 384 位曲线阶而言较小。令 $B=2^{340}$，用以下格基编码 $m-1$ 组差分，其中前 $m-1$ 行是对角线上的 $n$，倒数第二行放 $t'_i$，最后一行放 $a'_i$：

$$
L=
\begin{bmatrix}
n&&&&0&0\\
&n&&&0&0\\
&&\ddots&&\vdots&\vdots\\
t'_1&t'_2&\cdots&t'_{m-1}&B/n&0\\
a'_1&a'_2&\cdots&a'_{m-1}&0&B
\end{bmatrix}.
$$

LLL 约化后，末坐标为 $B$ 的短向量包含 $Bd/n$，从而恢复私钥。核心 Sage 函数如下：

```sage
def recover_private_key(n, hashes, signatures, B=2^340, low_bits=25):
    Zn = Zmod(n)
    count = len(signatures) - 1

    basis = [
        [0] * i + [n] + [0] * (count - i - 1 + 2)
        for i in range(count)
    ]

    ts = [Zn(-r2 / ((2^low_bits) * Zn(s)))
          for r1, r2, s in signatures]
    basis.append([ZZ(ts[0] - value) for value in ts[1:]] + [B / n, 0])

    aa = [Zn(h * r1 / ((2^low_bits) * Zn(s)))
          for h, (r1, r2, s) in zip(hashes, signatures)]
    basis.append([ZZ(aa[0] - value) for value in aa[1:]] + [0, B])

    for row in Matrix(basis).LLL():
        if row[-1] == B:
            return Zn(row[-2] * n / B)
    raise ValueError("private key not found")
```

恢复 $d$ 后可以自行选择两个非零 nonce，按服务公式生成目标消息签名：

```sage
import hashlib
import ecdsa

Curve = ecdsa.NIST384p
G = Curve.generator
n = Curve.order
message = b"I know alll of your secrets!"
h = Integer(int(hashlib.sha384(message).hexdigest(), 16))

k1, k2 = 133, 337
r1 = (k1 * G).x()
r2 = (k2 * G).y()
s = inverse_mod(k1, n) * (h * r1 - r2 * d) % n
print(r1, r2, s)
```

服务验证成功后返回：

```text
DUCTF{y0u_f0unD_7h3_h1dD3n_numB3r5!}
```

## 方法总结

签名 nonce 不必完全复用才会泄露私钥；固定低位、已知高位或其它偏差同样会形成隐藏数问题。本题中单个样本还带有未知公共后缀，先对样本做差消去公共项，再用 LLL 寻找由小 nonce 高位差构成的短向量，是决定性步骤。
