# DownUnderCTF 2022 cheap ring theory Writeup

## 题目简述

题目在商环

$$
Q=\mathbb F_p[x]/(f)
$$

中生成随机元素 $A,B$ 和秘密指数 $n,m$，并公开 $C=A^nB^m$。玩家需要提交三个向量 $\phi(A),\phi(B),\phi(C)\in\mathbb F_p^3$，服务逐坐标计算幂并检查：

$$
\phi(A)^n\odot\phi(B)^m=\phi(C).
$$

关键是公开三次多项式 $f$ 在 $\mathbb F_p$ 上完全分裂。

## 解题过程

设 $f(x)=(x-r_1)(x-r_2)(x-r_3)$。三个一次因子两两互素，由多项式环上的中国剩余定理：

$$
\mathbb F_p[x]/(f)\cong
\mathbb F_p[x]/(x-r_1)\times
\mathbb F_p[x]/(x-r_2)\times
\mathbb F_p[x]/(x-r_3).
$$

每个商环都通过“在根处求值”同构于 $\mathbb F_p$，因此可取：

$$
\phi(g)=(g(r_1),g(r_2),g(r_3)).
$$

求值保持加法和乘法，所以必然有 $\phi(A^nB^m)=\phi(A)^n\odot\phi(B)^m$。

```python
def phi(g):
    roots = [r for r, _ in f.roots()]
    lifted = g.lift()
    return vector(GF(p), [lifted(r) for r in roots])

send_vector(phi(A))
send_vector(phi(B))
send_vector(phi(C))
```

此外，校验没有要求提交值真的是输入的像。因为 $n,m\ge1$，直接令三个向量都为零，等式 $0^n0^m=0$ 也成立；全一向量同样能通过。这是更短的非预期解，但 CRT 映射解释了题目的预期结构。服务输出：

```text
DUCTF{CRT_e4sy_as_0ne_tw0_thr3e}
```

## 方法总结

当商多项式完全分裂为互素一次因子时，商环可以通过在各根处求值分解成有限域的直积，复杂的环乘法便转化为逐坐标运算。同时也要审查验证器是否把提交值绑定到题目输入；只校验代数恒等式而不校验来源时，零元或单位元常能形成退化解。
