# DownUnderCTF 2021 - yadlp

## 题目简述

题目定义了一种由二元组 $(x,y)\in\mathbb F_p^2$ 表示的自定义群，群运算为：

$$
(x_1,y_1)(x_2,y_2)=
(x_1x_2+Dy_1y_2,\ x_1y_2+x_2y_1+2y_1y_2).
$$

48 字节 flag 被切成 6 个 8 字节整数 $m_i$。题目公开随机群元素 $g_i$ 和：

$$
c=\prod_{i=1}^{6}g_i^{m_i}.
$$

目标是解决多重离散对数。关键在于识别该自定义群实际是有限域 $\mathbb F_{p^2}$ 中一个阶为 $p+1$ 的范数为 1 子群，而 $p+1$ 很光滑，可以用 Pohlig–Hellman 高效求离散对数。

## 解题过程

从 `rand_element` 的生成公式整理可得所有有效点都满足：

$$
x^2+2xy-Dy^2=1\pmod p.
$$

令 $W^2=D+1$。挑战参数满足 $D+1$ 在 $\mathbb F_p$ 中不是平方，因此
$\mathbb F_{p^2}=\mathbb F_p[W]/(W^2-(D+1))$。定义映射：

$$
\varphi(x,y)=(x+y)+yW.
$$

其范数为：

$$
((x+y)+yW)((x+y)-yW)
=(x+y)^2-(D+1)y^2=1.
$$

直接展开还能验证 $\varphi$ 保持题目定义的群运算。因此原群同构于 $\mathbb F_{p^2}^{\times}$ 中阶为 $q=p+1$ 的循环子群。

在 Sage 中建立扩域，取该阶子群的生成元并求所有离散对数：

```sage
exec(open('../challenge/output.txt').read())

F.<X> = GF(p)[]
R.<W> = GF(p^2, modulus=X^2 - (D + 1))
q = p + 1
generator = R.zeta(q)

def phi(point):
    x, y = point
    return (x + y) + y*W

logs = [discrete_log(phi(gi), generator) for gi in G]
target_log = discrete_log(phi(c), generator)
```

由于 $q$ 的素因子都较小，Sage 的离散对数会利用 Pohlig–Hellman 完成计算。群同态把乘积关系变成一个模线性方程：

$$
\sum_{i=1}^{6}m_iL_i\equiv L_c\pmod q,
$$

其中 $L_i=\log_g(g_i)$，$L_c=\log_g(c)$。未知量有 6 个，但每个 $m_i<2^{64}$，远小于约 512 位的 $q$，可用 LLL 找到小解。构造格基：

```sage
count = len(logs)
column = logs + [target_log, q]

B = Matrix.column(ZZ, vector(column))
B = B.augment(
    Matrix.identity(count + 1).stack(vector([0] * (count + 1)))
)

reduced = B.LLL()
for row in reduced:
    if row[0] == 0 and abs(row[-1]) == 1:
        blocks = [abs(Integer(v)) for v in row[1:-1]]
        break
```

该短向量对应 $(0,m_1,\ldots,m_6,-1)$，首坐标为零正是上面的模方程。把 6 个整数分别按 8 字节大端转换并连接：

```python
flag = b''.join(value.to_bytes(8, 'big') for value in blocks)
print(flag.decode())
```

得到：

```text
DUCTF{a_1337_hyp3rb0la_m33ts_th3_mult1pl3_DLP!!}
```

## 方法总结

遇到自定义二元群时，应从元素生成条件和运算公式推导不变量，再尝试映射到熟悉的有限域、椭圆曲线或整数群。本题的双曲线方程经变量替换成为 Pell 型范数方程，同构到 $\mathbb F_{p^2}$ 的范数 1 子群。离散对数解决后，多重 DLP 仍不是唯一线性方程直接求解，而是利用每个消息块很小的界，通过格约简恢复短整数向量。
