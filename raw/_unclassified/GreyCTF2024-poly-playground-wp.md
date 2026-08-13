# GreyCTF2024 Poly Playground WP

## 题目简述

服务给出 $1$ 到 $5$ 个整数根，每种根数量各出 20 题，要求提交首项系数为 1 的多项式系数。若根为 $r_1,\ldots,r_n$，目标多项式就是 $\prod_{i=1}^{n}(x-r_i)$。

## 解题过程

可以直接用 `numpy.poly` 展开给定根，也可以从系数 `[1]` 开始逐次卷积 `[1, -r]`。下面的实现只使用整数运算：

```python
def coefficients(roots):
    out = [1]
    for r in roots:
        nxt = [0] * (len(out) + 1)
        for i, a in enumerate(out):
            nxt[i] += a
            nxt[i + 1] -= a * r
        out = nxt
    return out
```

例如根为 $2,-3$ 时：

$$
(x-2)(x+3)=x^2+x-6
$$

因此提交 `1,1,-6`。自动读取服务的 `Roots:` 行、按逗号解析、计算并回传系数，完成全部 100 轮后得到：

```text
grey{l0oks_lik3_sOm3one_c4n_b3_a_po1ynomia1_w1z4rd}
```

## 方法总结

已知全部根时，构造首一多项式只是重复乘以一次因子 $(x-r)$。自己实现逐项更新可避免浮点误差，也能清楚保证输出顺序是从最高次项到常数项。
