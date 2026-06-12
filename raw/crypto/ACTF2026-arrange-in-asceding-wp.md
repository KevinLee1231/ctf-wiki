# ArrAnge in Asceding

## 题目简述

题目是 CKKS 近似同态加密应用。服务给出公钥和旋转密钥，随机生成长度为 128 的数组并加密；提交的密文被解密后，服务检查前 128 个 slot 是否等于原数组中每个元素的升序排名。解法是在密文域里完成所有两两比较并求和得到 rank。

## 解题过程

直接同态排序网络会消耗过深乘法层数。题目数组长度为 $128$，且每个值来自 $0..511$，CKKS 槽位数为：

$$
32768/2=16384=128\times128
$$

因此可以把所有两两比较输入打包到同一个密文的槽位中。设明文数组为 $B$，目标 rank 为：

$$
R_i=\sum_{j=1}^{128}[B_i>B_j]
$$

构造所有 offset 的差值：

```text
block offset, slot i: (B_i - B_{i+offset mod 128}) / 512
```

归一化后差值只落在离散集合：

```text
{-511/512, ..., -1/512, 0, 1/512, ..., 511/512}
```

所以不需要在连续区间上逼近阶跃函数，只需在这些离散点上拟合。实际脚本拟合的是比较函数：

$$
\mathrm{cmp}(x)=
\begin{cases}
1,&x>0\\
0.5,&|x|\le 10^{-5}\\
0,&x<0
\end{cases}
$$

由于题目用 `random.sample(range(512), 128)` 生成互不相同的数组，升序排名就是 $\sum_j[B_i>B_j]$。自比较会贡献 `0.5`，因此同态求和后再减去 `0.5` 即可：

$$
R_i=\sum_j\mathrm{cmp}(B_i-B_j)-0.5
$$

比较函数可以用 Taylor、Chebyshev 或 Remez 近似；这里使用 Chebyshev 系数拟合分段比较函数。实际 exp 中的比较函数为：

```python
def cmp_func(x):
    if x > 0.00001:
        return 1
    elif x > -0.00001:
        return 0.5
    else:
        return 0

poly = EvalChebyshevCoefficients(cmp_func, -1.0, 1.0, 512)
tmp = ps.eval_poly_ps(tmp, poly, evaluator, encoder, encryptor, relin_keys)
```

多项式求值不能直接用 Horner Rule，否则乘法深度过高；exp 里的 `ps.eval_poly_ps()` 按 Chebyshev 恒等式递归生成 $T_i(x)$，偶数项用 $T_{2n}=2T_n^2-1$，奇数项用 $T_{2n+1}=2T_nT_{n+1}-x$，并在每次密文乘法后重线性化、rescale：

```python
def cheb_term(index):
    if index in cheb_cache:
        return cheb_cache[index]

    if index % 2 == 0:
        half = cheb_term(index // 2)
        squared = mul(half, half)
        value = sub(add(squared, squared), const(squared, 1.0))
    else:
        left = cheb_term(index // 2)
        right = cheb_term(index // 2 + 1)
        product = mul(left, right)
        value = sub(add(product, product), enc)

    cheb_cache[index] = value
    return value
```

TenSEAL 还有一个实现细节：短 `CKKSVector` 的底层 slot 会周期性填充，不是只有前 128 个 slot 有值。直接手工构造重复 block 容易把差值布局放错。稳定做法是先将原始密文除以 512，再通过旋转构造 pairwise comparison 的两侧输入。`transform()` 负责把每个位置的值送到对应 block 的首槽，`RepC()` 再把首槽复制成和其他 128 个值比较所需的布局：

```python
def transform(enc):
    base = None
    for i in range(128):
        ct_new = sealapi.Ciphertext()
        evaluator.rotate_vector(enc, 128 * ((128 - i) % 128) + i, galois_keys, ct_new)
        if i == 0:
            base = ct_new
        else:
            evaluator.add_inplace(base, ct_new)
    mask = [1.0 if i % 128 == 0 else 0.0 for i in range(32768 // 2)]
    encoder.encode(mask, 2**40, plain_mul)
    evaluator.multiply_plain_inplace(base, plain_mul)
    evaluator.rescale_to_next_inplace(base)
    return base

def RepC(enc):
    base = sealapi.Ciphertext()
    evaluator.rotate_vector(enc, 0, galois_keys, base)
    for i in range(1, 128):
        ct_new = sealapi.Ciphertext()
        evaluator.rotate_vector(enc, i + 128 * 127, galois_keys, ct_new)
        evaluator.add_inplace(base, ct_new)
    return base
```

对所有 block 的比较结果按 128 的步长旋转求和后，减去 `0.5` 抵消自比较的中点贡献。服务端 `round()` 后得到 rank。

远程交互带 kCTF Sloth PoW，脚本从 banner 中提取 `solve s.xxx.yyy` 并本地反算。提交时必须提交原生 SEAL ciphertext 的序列化结果，因为服务端加载方式是：

```python
sealapi.Ciphertext().load(ctxdata, "ct.bin")
```

目录中需要放 `ctx.public` 和 `galois.key`；如果只有 `pubkey.zip`，先解压得到这两个文件。最终运行方式：

```bash
python3 exploit.py --host <题目环境地址> --root .
```

提交通过后，服务端会输出题目 flag；本地附件中的 flag 值为 redacted，因此正文只保留触发条件和算法主线，不臆造具体 flag。

脚本最终输出解出的排列和 flag，验证了排序约束与解密流程。

## 方法总结

- 核心技巧：把 ranking 转成 $128\times128$ 个 SIMD 两两比较，拟合 $0.5\cdot\mathrm{sign}(x)$ 后求和得到 rank。
- 识别信号：CKKS 题给出旋转密钥、slot 数足够容纳所有 pairwise comparison 时，应优先考虑 SIMD 打包和旋转求和。
- 复用要点：先归一化离散差值，再选低深度 Chebyshev 求值方式；TenSEAL slot 周期填充和 SEAL ciphertext 序列化格式是复现时最容易踩的工程点。
