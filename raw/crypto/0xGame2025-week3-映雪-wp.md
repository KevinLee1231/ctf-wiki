# 映雪

## 题目简述

题目在复合模数 $n=pq$ 上使用椭圆曲线

$$
E:\ y^2=x^3+px+q\pmod n
$$

并用标量 $e=65537$ 乘明文点。服务端给出曲线上的两个点和 $n$，目标是先从曲线方程恢复因子 $p,q$，再分别在 $\mathbb F_p$ 与 $\mathbb F_q$ 上逆转标量乘法，最后通过 CRT 合并明文点。

原 PDF 将其称为“双线性配对”，但给出的求解脚本并没有计算 pairing；决定解法的是复合模椭圆曲线的 CRT 分解和群阶上的标量求逆。

## 解题过程

设两个公开点为 $(x_1,y_1)$、$(x_2,y_2)$。两式相减可以消去常数项 $q$：

$$
y_1^2-y_2^2\equiv x_1^3-x_2^3+p(x_1-x_2)\pmod n.
$$

当 $x_1-x_2$ 在模 $n$ 下可逆时：

$$
p\equiv\big((y_1^2-y_2^2)-(x_1^3-x_2^3)\big)(x_1-x_2)^{-1}\pmod n.
$$

题目恰好把素因子 $p$ 用作曲线系数，因此该值直接整除 $n$，进而得到 $q=n/p$。

模 $p$ 时曲线化为 $E_p:y^2=x^3+q$；模 $q$ 时化为 $E_q:y^2=x^3+px$。若密文点满足 $C=eM$，则分别计算：

$$
d_p\equiv e^{-1}\pmod{\#E_p},\qquad
d_q\equiv e^{-1}\pmod{\#E_q},
$$

并求 $M_p=d_pC_p$、$M_q=d_qC_q$。最后对两个坐标分别做 CRT。

```sage
from sage.all import EllipticCurve, Zmod, ZZ, crt

x1, y1 = ...
x2, y2 = ...
n = ...
e = 65537

p = ZZ(((y1**2 - y2**2) - (x1**3 - x2**3))
       * pow(x1 - x2, -1, n) % n)
assert n % p == 0
q = n // p

Ep = EllipticCurve(Zmod(p), [0, q])
Cp = Ep(x1 % p, y1 % p)
Mp = pow(e, -1, Ep.order()) * Cp
xp, yp = map(ZZ, Mp.xy())

Eq = EllipticCurve(Zmod(q), [p, 0])
Cq = Eq(x1 % q, y1 % q)
Mq = pow(e, -1, Eq.order()) * Cq
xq, yq = map(ZZ, Mq.xy())

xm = crt([xp, xq], [p, q])
ym = crt([yp, yq], [p, q])
E = EllipticCurve(Zmod(n), [p, q])
E(xm, ym)  # 验证明文点确实落在原曲线上

answer = int(xm).to_bytes(126, 'big')[:35]
print(answer)
```

题目把待恢复内容放在明文点 $x$ 坐标的高 35 字节；打印 `answer` 或按交互要求提交其十六进制即可完成验证。

## 方法总结

- 同一短 Weierstrass 曲线上的两个点可以通过方程相减消去常数项，并恢复一次项系数。
- 当曲线系数本身就是 $n$ 的素因子时，恢复系数等价于分解复合模数。
- 复合模上的椭圆曲线运算应拆到 $\mathbb F_p$、$\mathbb F_q$ 两个域中处理；分别逆标量乘法后，再用 CRT 合并坐标。
