# S0DH

## 题目简述

题目使用 SIDH 风格的超奇异同源 Diffie–Hellman。基础域和曲线参数为

$$
p=2^{38}3^{25}-1,\qquad
E_0/\mathbb F_{p^2}:y^2=x^3+6x^2+x.
$$

题目给出 $E_0$ 上的 $2^{38}$ 阶基 $(P_a,Q_a)$、$3^{25}$ 阶基 $(P_b,Q_b)$，以及 Alice、Bob 的公钥曲线 $E_a,E_b$。Alice 的秘密核具有

$$
\langle P_a+s_aQ_a\rangle
$$

的形式，Bob 还公开了 $\phi_b(P_a)$ 和 $\phi_b(Q_a)$。密文是明文整数与共享曲线 $j$-不变量哈希的异或：

$$
c=m\oplus
\operatorname{SHA256}\bigl(\operatorname{str}(j(E_{ab}))\bigr).
$$

由于 Alice 侧的同源次数只有 $2^{38}$，可以把同源路径从中间拆开，执行 meet-in-the-middle，而不必穷举全部 $2^{38}$ 个秘密。

## 解题过程

### 拆分同源路径

取

$$
t=\left\lfloor\frac{38}{2}\right\rfloor+1=20.
$$

从 $E_0$ 一侧枚举所有次数为 $2^t$ 的循环子群。对每个候选核计算商曲线，并以商曲线的 $j$-不变量为键建立哈希表：

```sage
steps = a // 2 + 1

def build_table():
    table = {}
    P = 2**(a - steps) * Pa
    Q = 2**(a - steps) * Qa

    kernel = P
    for s in range(2**steps):
        j_middle, _ = isog_2k(A0, kernel, steps)
        table[j_middle] = s
        kernel += Q
    return table
```

`isog_2k` 使用 Montgomery 曲线的投影 $x$ 坐标，按核点依次计算 2-同源和 4-同源。枚举时只需要商曲线的 $j$-不变量，因此无须在每一步恢复完整的仿射点；这是让约 $2^{20}$ 次枚举可行的关键实现优化。

### 从 $E_a$ 反向搜索

从 $E_a$ 选择一组完整的 $2^{38}$ 阶基 $(P'_a,Q'_a)$，然后枚举剩余 $2^{18}$ 阶同源。循环子群不能只覆盖 SIDH 常见的

$$
\langle P'_a+sQ'_a\rangle;
$$

还要覆盖另一族

$$
\langle tP'_a+Q'_a\rangle,
$$

否则会漏掉射影直线中的一个方向。每个候选同样计算中间曲线 $j$-不变量，并与前向表碰撞：

```sage
def search_from_Ea(table, remaining_steps):
    collisions = []
    P = 2**(a - remaining_steps) * Pa_on_Ea
    Q = 2**(a - remaining_steps) * Qa_on_Ea

    kernel = P
    for s in range(2**remaining_steps):
        result = isog_2k(Ea_A, kernel, remaining_steps)
        if result is not None and result[0] in table:
            collisions.append((1, s, table[result[0]]))
        kernel += Q

    kernel = Q
    for t in range(0, 2**remaining_steps, 2):
        result = isog_2k(Ea_A, kernel, remaining_steps)
        if result is not None and result[0] in table:
            collisions.append((t, 1, table[result[0]]))
        kernel += 2 * P

    return collisions
```

两个方向命中的同一 $j$-不变量说明找到了同构的中间曲线。

### 重建同源并恢复秘密

设碰撞给出的前向核为 $K_1$、从 $E_a$ 反向枚举的核为 $K_2$。用 Sage 重建两段同源：

```sage
phi1 = E0.isogeny(
    kernel=K1,
    algorithm="factored",
    model="montgomery",
)
phi2 = Ea.isogeny(
    kernel=K2,
    algorithm="factored",
    model="montgomery",
)

middle_1 = phi1.codomain()
middle_2 = phi2.codomain()
assert middle_1.j_invariant() == middle_2.j_invariant()

sigma = middle_1.isomorphism_to(middle_2)
phi = phi2.dual() * sigma * phi1
```

完整映射 $\phi:E_0\to E_a$ 的核包含 $P_a+s_aQ_a$，所以

$$
\phi(P_a)+s_a\phi(Q_a)=\mathcal O.
$$

在 $2^{38}$ 阶子群中做一次离散对数即可恢复

```sage
phi_P = phi(Pa)
phi_Q = phi(Qa)
sa = -phi_Q.discrete_log(phi_P)
sa %= 2**a
```

最后把恢复的 $s_a$ 作用到 Bob 公开的基像上：

```sage
Eab = Eb.isogeny(
    kernel=phib_Pa + sa * phib_Qa,
    algorithm="factored",
    model="montgomery",
).codomain()

jab = Eab.j_invariant()
mask = bytes_to_long(hashlib.sha256(str(jab).encode()).digest())
flag = long_to_bytes(mask ^ enc)
```

官方材料只给出求解脚本，没有附最终打印出的 flag，因此不补写未经当前材料验证的结果。

## 方法总结

- 核心技巧：把 $2^{38}$-同源路径分成两段，以中间曲线的 $j$-不变量做 meet-in-the-middle，复杂度降到约 $2^{20}$ 级别。
- 识别信号：SIDH 风格参数中某一侧同源次数明显偏小，并公开起点、终点曲线时，应检查能否从两端枚举到同一同构类。
- 复用要点：哈希键应使用同构不变量而不是曲线方程系数；反向枚举要覆盖射影参数的两族核；碰撞后仍需通过对偶同源和曲线同构重建完整映射，再从核关系恢复秘密。
