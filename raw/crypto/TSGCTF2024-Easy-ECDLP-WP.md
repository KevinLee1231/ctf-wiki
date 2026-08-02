# TSGCTF2024 Easy? ECDLP

## 题目简述

题目在环 $\mathbb Z/p^4\mathbb Z$ 上建立椭圆曲线 $E:y^2=x^3+ax+b$，给出基点 $P$ 和：

$$Q=\text{secret}\cdot P$$

`secret` 由 flag 后接随机字节组成，总长 127 字节，因此不能只恢复短 flag 前缀。参数经过刻意选择，使曲线约化到 $\mathbb F_p$ 后满足：

$$\#E(\mathbb F_p)=p$$

这是一条 anomalous curve，可用 Smart–Satoh–Araki 思路把 ECDLP 转到 $p$ 进形式群。

## 解题过程

先把环上坐标提升到精度 4 的 $p$ 进域：

```sage
k = Qp(p, 4)
E = EllipticCurve(k, [a, b])
P = E.point((X, Y, Z))
Q = E.point((XQ, YQ, 1))

E0 = EllipticCurve(GF(p), [a, b])
assert E0.order() == p
```

因为约化曲线阶为 $p$，对任意可约化点乘以 $p$ 后，$pP$ 与 $pQ$ 都落入约化映射的核，也就是椭圆曲线形式群。形式群对数 $\log_E$ 在这里把点加法线性化：

$$\log_E([n]R)=n\log_E(R)$$

Sage 给出形式对数的截断多项式。以本题坐标约定代入参数 $t=x/y$：

```sage
ell = E.formal_group().log().polynomial()

def formal_log(R):
    return ell.subs(t=R[0] / R[1])
```

由 $pQ=\text{secret}\cdot pP$ 得到：

$$
\text{secret}\equiv
\frac{\log_E(pQ)}{\log_E(pP)}
$$

在有限 $p$ 进精度下先取其整数代表：

```sage
ans = ZZ(formal_log(p * Q) / formal_log(p * P))
```

该结果已经恢复 secret 的主要 $p$ 进位数，但由于 `formal_log(pP)` 自带正 valuation，除法会损失一部分绝对精度。计算剩余点：

$$R=Q-[\text{ans}]P$$

它对应尚未恢复的倍数差，并位于形式群中。又因为 $\log_E(pP)=p\log_E(P)$，有：

$$
\text{secret}-\text{ans}
=p\cdot\frac{\log_E(R)}{\log_E(pP)}
$$

于是做第二次 Hensel 式修正：

```sage
R = Q - ans * P
ans += ZZ(formal_log(R) / formal_log(p * P) * p)
assert ans * P == Q
```

把恢复的整数按大端转回字节：

```sage
from Crypto.Util.number import long_to_bytes
print(long_to_bytes(int(ans)))
```

输出开头为：

```text
TSGCTF{HeNSel's L3mMa 1s s0 usefUl!}
```

其后的字节是生成题目时附加的随机填充，flag 在右花括号处结束。

## 方法总结

本题不是在通用 1024 位群上硬做 ECDLP，而是利用 anomalous reduction：$\#E(\mathbb F_p)=p$ 使 $pP,pQ$ 进入可线性化的形式群，形式对数之比直接给出标量的 $p$ 进信息。有限精度下第一次比值不足以覆盖 127 字节 secret，再对残差点做一次 Hensel 提升即可补齐。看到定义在 $\mathbb Z/p^k\mathbb Z$ 上且约化曲线阶异常的 ECDLP，应优先检查形式群，而不是直接套普通有限域离散对数。
