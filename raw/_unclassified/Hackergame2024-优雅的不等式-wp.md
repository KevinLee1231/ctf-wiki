# 优雅的不等式

## 题目简述

服务连续给出 40 个有理数 $p/q\leq\pi$，要求提交长度不超过 400、只含整数、`x` 和加减乘除幂运算的函数 $f(x)$，并同时通过两个精确检查：

$$
\int_0^1 f(x)\,dx=\pi-\frac pq,
\qquad f(x)\geq 0\quad(x\in[0,1]).
$$

前一档固定证明 $\pi\geq 8/3$；后一档的 $q$ 随轮次指数增长，界会逼近 $\pi$。决定性障碍是构造可被 SymPy 精确积分和证明非负的分析函数，而不是密码原语，因此归入 `_unclassified`。

## 解题过程

### 第一档：圆弧与抛物线之间的面积

单位四分之一圆弧为 $\sqrt{1-x^2}$，抛物线 $1-x^2$ 在 $[0,1]$ 上位于圆弧下方，而且

$$
4\int_0^1\sqrt{1-x^2}\,dx=\pi,
\qquad
4\int_0^1(1-x^2)\,dx=\frac83.
$$

因此直接提交

```text
4*((1-x**2)**(1/2)-(1-x**2))
```

其积分正好是 $\pi-8/3$，非负性也由 $0\leq1-x^2\leq1$ 推出 $\sqrt{1-x^2}\geq1-x^2$。

### 后续各档：构造任意精确差值

使用函数族

$$
h_n(x)=\frac{(x(1-x))^n}{1+x^2}.
$$

分子在 $[0,1]$ 上非负，分母恒正。对 $n$ 为 8 的倍数时，SymPy 会把积分整理成

$$
I_n=\int_0^1h_n(x)\,dx=\alpha_n\pi+\beta_n,
$$

其中 $\alpha_n>0$、$\beta_n<0$ 均为有理数。令

$$
g_n(x)=\frac{h_n(x)}{\alpha_n},
\qquad
\int_0^1g_n(x)\,dx=\pi+r_n,
$$

这里 $r_n=\beta_n/\alpha_n<0$。另取

$$
g_0(x)=\frac4{1+x^2},
\qquad \int_0^1g_0(x)\,dx=\pi.
$$

对当前目标 $p/q$，设

$$
w=\frac{p/q}{-r_n}.
$$

只要选取足够大的 $n$ 使 $-r_n\geq p/q$，就有 $0\leq w\leq1$。凸组合

$$
f(x)=w g_n(x)+(1-w)g_0(x)
$$

保持非负，并满足

$$
\int_0^1f(x)\,dx=w(\pi+r_n)+(1-w)\pi
=\pi-\frac pq.
$$

官方取 $n=80$ 已足以覆盖所有轮次。生成提交表达式的核心代码如下；其中所有系数都由 SymPy 保持为精确有理数：

```python
import sympy as sp

x = sp.Symbol("x")
n = 80
den = 1 + x**2
h = (x * (1 - x))**n
I = sp.integrate(h / den, (x, 0, 1))
alpha = sp.expand(I).coeff(sp.pi)
h = h / alpha
I = I / alpha

def answer(p, q):
    w = sp.Rational(p, q) / (sp.pi - I)
    f = sp.simplify((w * h + (1 - w) * 4) / den)
    assert sp.integrate(f, (x, 0, 1)) == sp.pi - sp.Rational(p, q)
    text = str(f).replace(" ", "")
    assert len(text) <= 400
    assert set(text) <= set("0123456789+-*/()x") and "//" not in text
    return text
```

因 $w\in[0,1]$ 且 $g_n,g_0\geq0$，服务端的区间非负性求解也能快速结束。

## 方法总结

- 核心技巧：寻找积分为“有理数乘 $\pi$ 加有理数”的非负函数族，再用两个非负函数的凸组合精确匹配任意目标差值。
- 识别信号：校验器要求符号积分精确等于 $\pi-r$，同时限制表达式长度和非负性时，应优先选择带 $1+x^2$ 分母、积分自然产生反正切与 $\pi$ 的函数。
- 复用要点：浮点近似不能通过结构相等检查，所有系数必须是精确有理数；不仅要匹配积分，还要保证权重落在 $[0,1]$，这样非负性才能由凸组合直接得到。
