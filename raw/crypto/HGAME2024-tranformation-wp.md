# tranformation

## 题目简述

题目使用有限域上的广义 Edwards 曲线：

$$
x^2+y^2\equiv c^2\left(1+d x^2y^2\right)\pmod p.
$$

曲线参数 $p,c,d$ 没有直接给出，只提供四个曲线上点 $P,Q,S,T$，以及公开指数 $e=65537$ 和点 $eG$。目标分两步：先由四个点消元恢复曲线参数，再把 Edwards 曲线双有理映射到 Sage 支持的短 Weierstrass 曲线，在同构曲线群上计算 $G=e^{-1}(eG)$，最后映射回原曲线并按题目编码得到 flag。

## 解题过程

### 由四个点恢复模数与曲线参数

对曲线上任意两点 $(x_i,y_i)$、$(x_j,y_j)$，两条曲线方程相减：

$$
(x_i^2-x_j^2)+(y_i^2-y_j^2)
-c^2d(x_i^2y_i^2-x_j^2y_j^2)\equiv0\pmod p.
$$

定义：

$$
A_{ij}=(x_i^2-x_j^2)+(y_i^2-y_j^2),
$$

$$
B_{ij}=x_i^2y_i^2-x_j^2y_j^2.
$$

则：

$$
A_{ij}\equiv c^2dB_{ij}\pmod p.
$$

用两组点对消去 $c^2d$，可得：

$$
A_{ij}B_{kl}-A_{kl}B_{ij}\equiv0\pmod p.
$$

所以多个交叉乘积都含因子 $p$，对它们取 GCD，再分解并选择能使四个点全部满足方程的大素因子即可恢复 $p$。随后：

$$
c^2d\equiv A_{12}B_{12}^{-1}\pmod p,
$$

$$
c^2\equiv x_1^2+y_1^2-(c^2d)x_1^2y_1^2\pmod p,
$$

$$
d\equiv(c^2d)(c^2)^{-1}\pmod p.
$$

下面是按官方 PDF 整理的参数恢复代码。原代码直接取 `factor(...)[1][0]`；更稳妥的做法是枚举 GCD 的素因子，并用四个点回代验证。

```python
def is_on_curve(curve, point):
    c, d, p = curve
    x, y = point
    return (
        x**2 + y**2 - c**2 * (1 + d * x**2 * y**2)
    ) % p == 0


def a_and_b(x1, x2, y1, y2):
    a = x1**2 - x2**2 + y1**2 - y2**2
    b = x1**2 * y1**2 - x2**2 * y2**2
    return a, b


def modulus_multiple(points):
    (x1, y1), (x2, y2), (x3, y3), (x4, y4) = points
    a12, b12 = a_and_b(x1, x2, y1, y2)
    a13, b13 = a_and_b(x1, x3, y1, y3)
    a23, b23 = a_and_b(x2, x3, y2, y3)
    a24, b24 = a_and_b(x2, x4, y2, y4)
    return gcd(
        a12 * b13 - a13 * b12,
        a23 * b24 - a24 * b23,
    )


P = (
    423323064726997230640834352892499067628999846,
    44150133418579337991209313731867512059107422186218072084511769232282794765835,
)
Q = (
    1033433758780986378718784935633168786654735170,
    2890573833121495534597689071280547153773878148499187840022524010636852499684,
)
S = (
    875772166783241503962848015336037891993605823,
    51964088188556618695192753554835667051669568193048726314346516461990381874317,
)
T = (
    612403241107575741587390996773145537915088133,
    64560350111660175566171189050923672010957086249856725096266944042789987443125,
)
points = [P, Q, S, T]

multiple = modulus_multiple(points)
print(factor(multiple))

# 从分解结果中选取能通过全部回代检查的大素因子。
p = 67943764351073247630101943221474884302015437788242536572067548198498727238923

a12, b12 = a_and_b(P[0], Q[0], P[1], Q[1])
c_squared_d = a12 * inverse_mod(b12, p) % p
c_squared = (
    P[0]**2
    + P[1]**2
    - c_squared_d * P[0]**2 * P[1]**2
) % p
d = c_squared_d * inverse_mod(c_squared, p) % p

field = GF(p)
c = field(c_squared).sqrt()
curve = (c, d, p)
assert all(is_on_curve(curve, point) for point in points)

print("p =", p)
print("c =", c)
print("d =", d)
```

恢复结果为：

```text
p = 67943764351073247630101943221474884302015437788242536572067548198498727238923
c = 7143899698109428282870539364581968579753042129945786627292343174759297201080
d = 8779982120820562807260290996171144226614358666469579196351820160975526615300
```

平方根存在正负两个候选。应继续用公开点关系及最终 flag 格式验证分支，而不能只看到四个点满足含 $c^2$ 的曲线方程就认为符号已经唯一确定。

### Edwards 映射到 Montgomery

先令：

$$
x'=\frac{x}{c},\qquad y'=\frac{y}{c},\qquad D=dc^4,
$$

原方程变为标准化 Edwards 形式：

$$
x'^2+y'^2=1+Dx'^2y'^2.
$$

再定义：

$$
u=\frac{1+y'}{1-y'},\qquad
v=\frac{2(1+y')}{x'(1-y')}=\frac{2u}{x'},
$$

$$
A=\frac{4}{1-D}-2,\qquad B=\frac{1}{1-D}.
$$

得到 Montgomery 曲线：

$$
Bv^2=u^3+Au^2+u.
$$

### Montgomery 映射到短 Weierstrass

令：

$$
X=\frac{u}{B}+\frac{A}{3B},\qquad Y=\frac{v}{B},
$$

则得到 Sage 可直接处理的短 Weierstrass 曲线：

$$
Y^2=X^3+aX+b,
$$

其中：

$$
a=\frac{3-A^2}{3B^2},\qquad
b=\frac{2A^3-9A}{27B^3}.
$$

官方 PDF 的说明行把这一式排成了 $X^3+aX^2+b$，但紧随其后的 `EllipticCurve(F, [a, b])` 和上述代换都对应 $X^3+aX+b$；实算回代也只对后者成立。因此正文采用经过代数与代码共同验证的形式。

### 在同构曲线上求逆标量并映射回来

完整 Sage 验证代码如下：

```python
p = 67943764351073247630101943221474884302015437788242536572067548198498727238923
c = 7143899698109428282870539364581968579753042129945786627292343174759297201080
d = 8779982120820562807260290996171144226614358666469579196351820160975526615300
e = 0x10001
eG = (
    40198712137747628410430624618331426343875490261805137714686326678112749070113,
    65008030741966083441937593781739493959677657609550411222052299176801418887407,
)

F = GF(p)
c = F(c)
D = F(d) * c**4
gx = F(eG[0]) / c
gy = F(eG[1]) / c
assert gx**2 + gy**2 == 1 + D * gx**2 * gy**2


def edwards_to_montgomery(x, y):
    u = (1 + y) / (1 - y)
    v = 2 * (1 + y) / (x * (1 - y))
    return u, v


A = 4 / (1 - D) - 2
B = 1 / (1 - D)
u, v = edwards_to_montgomery(gx, gy)
assert B * v**2 == u**3 + A * u**2 + u


def montgomery_to_weierstrass(u, v):
    X = u / B + A / (3 * B)
    Y = v / B
    return X, Y


a = (3 - A**2) / (3 * B**2)
b = (2 * A**3 - 9 * A) / (27 * B**3)
E = EllipticCurve(F, [a, b])
X, Y = montgomery_to_weierstrass(u, v)
encrypted_point = E(X, Y)

order = E.order()
assert gcd(e, order) == 1
plain_point = inverse_mod(e, order) * encrypted_point


def weierstrass_to_edwards(point):
    X, Y = point[0], point[1]
    u = (X - A / (3 * B)) * B
    v = B * Y
    assert B * v**2 == u**3 + A * u**2 + u

    x_normalized = 2 * u / v
    y_normalized = (u - 1) / (u + 1)
    assert (
        x_normalized**2 + y_normalized**2
        == 1 + D * x_normalized**2 * y_normalized**2
    )
    return x_normalized * c, y_normalized * c


plain_x, plain_y = weierstrass_to_edwards(plain_point)
plain_x = int(plain_x)
plain_y = int(plain_y)

print("order =", order)
print("plain_x =", plain_x)
print("plain_y =", plain_y)
print("hgame{" + hex(plain_x + plain_y)[2:] + "}")
```

本地 Sage 实算得到：

```text
order = 67943764351073247630101943221474884302071957392340923189748226436548954126268
plain_x = 10801522842243173004305732551018051267087389767241338575531365181016273121234
plain_y = 45542712889400624552765069228326432314004665232870865493507801651803120421882
```

官方 PDF 只打印恢复后的点，没有附明文编码规则。[出题人赛后公开的题目源码与说明](https://pythok.icu/2024/03/20/2024-Hgame-week4-wp-crypto/)给出了缺失的定义：

```python
flag = "hgame{" + hex(gx + gy)[2:] + "}"
```

因此最终结果为：

```text
hgame{7c91b51150e2339628f10c5be61d49bbf9471ef00c9b94bb0473feac06303bcc}
```

## 方法总结

- 未知 Edwards 参数可先通过多点方程相减消去常数项，再用交叉乘法消去 $c^2d$；多个结果取 GCD 即可暴露模数因子。
- 从 GCD 分解结果选择 $p$ 时必须把所有已知点回代，不能依赖因子列表的固定下标。
- Sage 不直接提供本题广义 Edwards 模型的群阶运算时，可以依次标准化并映射到 Montgomery、短 Weierstrass 模型；双有理映射前后都应逐步断言点满足对应曲线。
- 由 $eG$ 恢复 $G$ 不是离散对数：已知 $e$ 且 $gcd(e,\#E)=1$ 时，只需计算 $e^{-1}\bmod\#E$ 并做一次标量乘。
- 公式文本、Sage 构造器和代数推导出现冲突时，应以回代验证定真伪。本题短 Weierstrass 项应为 $aX$，不是 PDF 说明中的 $aX^2$。
- 曲线点恢复后仍需题目定义的编码规则。这里 flag 来自 `hex(gx + gy)`，直接对单个坐标做 `long_to_bytes` 会得到不可打印数据。
