# GreyCTF2022 - Equation

## 题目简述

题目把 flag 分成两个大整数 $m_1,m_2$，只给出两条二元整系数多项式方程。未知量虽然大，但方程次数低，可以用结式消元把二元问题降成一元多项式求根。

## 解题过程

根据生成脚本建立多项式环，并把题目给出的常数移到左侧：

```sage
R.<m1, m2> = PolynomialRing(ZZ)
f = 13*m2**2 + m1*m2 + 5*m1**7 - C1
g = 7*m2**3 + m1**5 - C2
```

对 $m_2$ 取结式会消去该变量：

$$R_1(m_1)=\operatorname{Res}_{m_2}(f,g).$$

分解 $R_1$ 或求其整数根，逐个代回原方程筛选；再以同样方法或直接代入求出 $m_2$。最后把两个整数分别转为大端字节并拼接：

```sage
r = f.resultant(g, m2)
for m1_value, _ in r.roots(ZZ):
    # 代回并检查两条原方程
    ...
print(long_to_bytes(m1_value) + long_to_bytes(m2_value))
```

在固定 `sage` 环境运行仓库中的官方 `sol.sage`，输出为：

```text
grey{solving_equation_aint_that_hard_rite_gum0pX6XzA5PJuro}
```

## 方法总结

多元低次方程优先考虑 Gröbner 基或结式。结式可能带来伪根和重根，最终必须把候选代回所有原方程，并用 flag 长度、字节可打印性等约束做二次确认。
