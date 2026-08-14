# i luv nonlinear

## 题目简述

题目把 32 字节 flag 作为整数 $x$，在模合数 $p$ 下递推 8 次 `ct = ct*x + r_i`。固定随机种子使常数 $r_i$ 已知，因此密文等价于一个关于 $x$ 的八次多项式同余。目标是利用 $p$ 的因数结构、Hensel 提升和中国剩余定理恢复 $x$。

## 解题过程

给定模数可以分解为：

$$p=q_1^2q_2^2$$

其中：

```text
q1 = 18446744073709551557
q2 = 18446744073709551629
```

用同样的 `random.seed(0)` 生成常数，把递推展开为整数多项式 $f(x)$，要求：

$$f(x)-c\equiv0\pmod p$$

先分别在 $q_1$、$q_2$ 的 $p$ 进域上求根。Sage 的 `Zp(q)` 多项式根会将模素数根提升到所需精度；取模 $q_i^2$ 后得到两组候选：

```sage
q1, q2 = [q for q, _ in factor(p)]

K.<x> = PolynomialRing(Zp(q1))
r1 = [ZZ(root[0] % (q1^2)) for root in (enc(x) - ct).roots()]

K.<x> = PolynomialRing(Zp(q2))
r2 = [ZZ(root[0] % (q2^2)) for root in (enc(x) - ct).roots()]
```

最后枚举两侧根的组合，并用 CRT 合并到模 $p$ 的解：

```sage
for a in r1:
    for b in r2:
        candidate = crt([a, b], [q1^2, q2^2])
        raw = bytes.fromhex(hex(candidate)[2:])
        if raw.startswith(b"grey{") and raw.endswith(b"}"):
            print(raw)
```

满足 flag 格式的唯一候选为：

```text
grey{crt_4nd_h4ns31_l1ft1ng_upz}
```

## 方法总结

合数模上的多项式求根可先拆到各素数幂模数：模素数求根、Hensel 提升到素数幂，再用 CRT 合并。固定 PRNG 种子让所谓随机系数变为公开常量，而 flag 格式可用于从多个代数根中筛选正确解。
