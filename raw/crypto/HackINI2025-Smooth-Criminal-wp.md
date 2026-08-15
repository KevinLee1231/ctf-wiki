# Smooth_Criminal

## 题目简述

题目在有限域 $\mathbb F_p$ 上使用椭圆曲线

$$
E:y^2=x^3+x,
$$

并给出基点 $G$ 与公钥点 $Q=dG$。私钥标量 $d$ 不是随机整数，而是把 flag 去掉 `shellmates{` 和 `}` 后的主体字节转成大整数。曲线名称与题目名都在提示：群阶 $p+1$ 被选成光滑数，离散对数可以用 Pohlig–Hellman 类方法快速求解。

## 解题过程

公开生成代码的关键部分为：

```python
F = GF(p)
E = EllipticCurve(F, [1, 0])
G = E.gens()[0]

body = FLAG.lstrip(b"shellmates{").rstrip(b"}")
priv = bytes_to_long(body)
Q = priv * G
```

本题 $p\equiv3\pmod4$，曲线 $y^2=x^3+x$ 的给定子群阶为 $p+1$。由于 $p+1$ 的素因子都足够小，Sage 的 `discrete_log` 会按群阶分解并组合各小子群中的离散对数，而不是在约 256 位群上做不可行的通用穷举。

重建曲线和点后直接求解：

```python
from sage.all import EllipticCurve, GF, discrete_log
from Crypto.Util.number import long_to_bytes

p = 88521243430530844680975622205504696165878102605019352642206406508236139194611
Gx = 33492873259653289851522652760138000432912970086603107614278819398809541678961
Gy = 56427944861700590310575747226418029847107876475483266465537727402293682324416
Qx = 7089509755460605801729915579661278437697783227562933747814580111886791675761
Qy = 28149764188091911667043999378653308588377636149074068910074771699114389734397

field = GF(p)
curve = EllipticCurve(field, [1, 0])
G = curve(Gx, Gy)
Q = curve(Qx, Qy)

assert G in curve and Q in curve
d = discrete_log(Q, G, operation="+", ord=p + 1)
assert d * G == Q

body = long_to_bytes(int(d))
print(body)
print(b"shellmates{" + body + b"}")
```

实际只读运行官方 Sage solver，恢复的标量为：

```text
14933578494406101768587808321688919656514047756979478259836473907
```

按大端字节转换得到主体 `$M0Oth_D0eS_Not_MEAN_S3cuR3`，补回题目生成时移除的包装后，flag 为：

```text
shellmates{$M0Oth_D0eS_Not_MEAN_S3cuR3}
```

## 方法总结

- 核心技巧：利用光滑群阶上的 Pohlig–Hellman，将椭圆曲线离散对数拆到多个小素数幂子群中求解。
- 识别信号：题目明确给出曲线、基点和 $Q=dG$，同时曲线名或参数暗示 `smooth`，应优先分解点阶或群阶，而不是直接尝试通用离散对数。
- 复用要点：传给 `discrete_log` 的 `ord` 必须是 $G$ 的阶或其可靠倍数；求出 $d$ 后用 $dG=Q$ 验证，再按生成代码的字节序和 flag 包装还原文本。
