# TinySEAL

## 题目简述

题目使用 BFV 全同态加密。服务给出随机明文多项式 $f(x)$ 的密文，要求提交只包含常数项 $a_0=f(0)$ 的密文，而服务端不向选手公开明文或私钥。

明文环为

$$
R=\operatorname{GF}(163841)[x]/(x^{4096}+1).
$$

可用操作包括密文加法、明文乘法和 Galois 自同构，但服务最多允许选择 12 个 Galois key。核心是利用全部奇数次自同构的“迹”消去非常数项，再用少量生成元复合出整个自同构群。

## 解题过程

对任意奇数 $j$，映射

$$
\theta_j(f(x))=f(x^j)
$$

是该分圆多项式商环上的自同构。可用的下标构成乘法群 $\mathbb Z_{8192}^{*}$，共有 $\varphi(8192)=4096$ 个元素。

把全部自同构结果相加：

$$
\operatorname{Tr}(f)=
\sum_{\substack{1\le j<8192\\j\text{ 为奇数}}}f(x^j).
$$

常数项在每个结果中都不变，所以累计为 $4096a_0$。对任意非常数单项式 $a_ix^i$，奇数指数遍历后会在模 $x^{4096}+1$ 的约化中两两异号，系数和为 0。因此

$$
\operatorname{Tr}(f)=4096a_0.
$$

最后同态乘以 $4096^{-1}\bmod 163841$，便得到只含 $a_0$ 的密文。

困难在于不能请求 4096 把 Galois key。自同构的复合满足

$$
\theta_a\circ\theta_b=\theta_{ab\bmod8192},
$$

所以只需请求一组生成元。经本地枚举可选：

```python
generators = [3, 5, 7, 11, 13, 17]
exponent_bound = 6
```

枚举每个指数向量 $(e_1,\ldots,e_6)\in\{0,\ldots,5\}^6$，计算

$$
j=\prod_{r=1}^{6}g_r^{e_r}\bmod8192,
$$

并为每个奇数 $j$ 保存总复合次数最少的指数向量。这个集合恰好覆盖全部 4096 个奇数剩余类，且最多只需要 6 把 key，低于题目限制。

收到原始密文后，对每个奇数下标重新加载一份密文，按保存的指数向量依次应用对应生成元的 Galois 变换，再累加：

```python
trace_ct = original_ct

for odd_index in range(3, 8192, 2):
    current = load_original_ciphertext()
    for generator, exponent in zip(generators, paths[odd_index]):
        for _ in range(exponent):
            evaluator.apply_galois_inplace(
                current, generator, galois_keys
            )
    evaluator.add_inplace(trace_ct, current)

scale = pow(4096, -1, 163841)
evaluator.multiply_plain_inplace(trace_ct, Plaintext(hex(scale)[2:]))
```

序列化 `trace_ct` 并提交，服务解密后看到的就是原多项式常数项，从而返回 flag。

完整的群生成搜索与 TenSEAL/SEAL API 实现见 [R3CTF 2024 Crypto Writeup](https://tang.cat/2024/06/10/R3CTF-2024-Crypto-Writeup.html)。正文已给出自同构迹为何提取常数项、如何压缩 Galois key 数量以及提交密文的全过程。

## 方法总结

这道题不是破解 BFV，而是在密文域内构造一个合法的线性算子。全部 Galois 自同构之和等价于环上的迹映射，会保留常数项并消掉其余项。题目的 12-key 限制需要进一步利用 $\mathbb Z_{8192}^{*}$ 的生成结构：密钥只对应生成元，其他自同构通过复合获得。实现时必须从原始密文分别生成每个群元素，不能在同一密文上连续变换后误把一条遍历路径当成独立求和项。
