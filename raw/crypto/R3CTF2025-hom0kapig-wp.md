# hom0kapig

## 题目简述

题目实现了一个精简 BFV 同态加密方案，参数为：

```text
N = 1024
p = 61441
q = 4123629569
B = 2
Delta = floor(q / p)
```

服务先给出一份随机明文的密文，随后允许选择两次操作，其中操作 1 会直接解密选手提交的任意密文并返回 SIMD 解码结果。最终只要提交的多项式与服务端三元秘密多项式 `sk` 完全相等即可得到 flag。

这相当于直接提供了选择密文解密 oracle；利用无需分析初始密文，也无需使用密钥切换或旋转密钥接口。

## 解题过程

### 观察解密与 SIMD 解码

密文由两个环元素 $(a,b)$ 组成，解密没有认证或合法性检查：

$$
m'=b+a\cdot sk \pmod{X^N+1,\ q}.
$$

SIMD 解码先把 $m'$ 的每个系数除以 $\Delta=\lfloor q/p\rfloor$ 并四舍五入到明文模数 $p$，再在预先排列的 $N$ 个单位根 `shift_roots` 上求值：

```python
encoded_poly = R([
    round(ZZ(c) / delta)
    for c in decrypted_poly.list()
])
return [encoded_poly(root) for root in shift_roots]
```

秘密多项式的每个系数都来自 $\{-1,0,1\}$。因此只要让解密结果恰好等于 $\Delta\cdot sk$，缩放和取整就会恢复 `sk` 在 $\mathbb F_p$ 上的系数；其中 $-1$ 对应 $p-1$。

### 一次查询泄露秘密多项式

提交如下伪造密文：

```text
a = Delta
b = 0
```

这里 `a` 是常数多项式。服务端计算：

$$
b+a\cdot sk=\Delta\cdot sk.
$$

对系数 $1$ 和 $0$，解码显然分别得到 $1$ 和 $0$。对系数 $-1$，模 $q$ 后的值是 $q-\Delta$，而题目参数满足：

$$
\operatorname{round}\left(\frac{q-\Delta}{\Delta}\right)=p-1.
$$

所以 oracle 返回的 1024 个槽值正是秘密多项式在 `shift_roots` 上的完整取值。

### 从槽值插值并提交

在 $\mathbb F_p$ 上按题目相同的根顺序做拉格朗日插值：

```python
R.<X> = PolynomialRing(GF(p))
sk_p = R.lagrange_polynomial([
    (shift_roots[i], leaked_slots[i])
    for i in range(N)
])
```

得到 `sk_p` 后，把系数 $p-1$ 还原成整数 $-1$，其余系数保持为 $0$ 或 $1$，再将多项式转换到 $\mathbb Z_q[X]/(X^{1024}+1)$。按照服务端 `recv_poly()` 接受的文本格式提交该多项式，即可通过：

```python
if secret_key == bfv.sk:
    print(flag)
```

整个过程只消耗一次解密查询，第二次操作可以不再利用。

## 方法总结

BFV 的解密接口不能直接暴露给不可信用户，更不能接受完全任意的 $(a,b)$。本题中令 $(a,b)=(\Delta,0)$，便把秘密多项式本身送入了明文缩放与 SIMD 解码流程；槽值又覆盖了多项式在全部根上的取值，因此一次插值即可无损恢复 `sk`。

核心不是破解 BFV 的困难问题，而是识别“选择密文解密 + 线性解密公式”带来的直接密钥恢复。题目部署文件中的占位 flag 也明确提示了方案缺少 IND-CCA 安全性，不能把解密 oracle 当成无害的测试功能。
