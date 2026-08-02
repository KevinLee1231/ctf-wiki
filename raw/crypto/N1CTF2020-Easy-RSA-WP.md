# N1CTF 2020 Easy RSA? Writeup

## 题目简述

题目把 RSA 与带小误差的线性系统叠在一起。两个 RSA 素数不是独立均匀生成，而是把若干 32 位系数代入基数 $x=3^{66}$ 的低次多项式得到；公钥指数为 $e=127$。flag 被编码成 43 维秘密向量 $s$，公开矩阵 $A\in\mathbb Z_q^{127\times43}$，每个明文满足：

$$
m_i=\langle A_i,s\rangle+b_i\pmod q,
$$

其中噪声 $b_i$ 服从标准差约 16 的窄分布，输出则为 $c_i=m_i^{127}\bmod N$。求解要依次完成结构化分解、RSA 解密和 LWE/CVP 去噪。

## 解题过程

### 把模数写成低次多项式

每个素数形如：

$$
p=P(x),\qquad q=Q(x),\qquad x=3^{66},
$$

$P,Q$ 的系数只有约 32 位。因此：

$$
N=pq=R(x)
$$

可看作一个约 8 次、系数远小于 $x$ 的多项式求值。直接按 $x$ 进制展开 $N$ 会有进位，不能简单切块。构造格，把 $N$ 与 $1,x,x^2,\ldots,x^8$ 的整数关系放入基矩阵，使用 LLL 找到系数较小且满足 $R(x)=N$ 的向量。恢复 $R(X)$ 后在整数多项式环分解：

```sage
R.<X> = PolynomialRing(ZZ)
factors = recovered_polynomial.factor()
p_candidate = factors[0][0](3^66)
p = gcd(N, p_candidate)
q = N // p
```

必须用 `gcd` 验证求值结果确实是 $N$ 的非平凡因子。

### RSA 解密取得带噪线性样本

求出 $p,q$ 后计算：

$$
d=e^{-1}\bmod (p-1)(q-1),\qquad m_i=c_i^d\bmod N.
$$

这些 $m_i$ 不是 flag 字节，而是 $A_is+b_i$ 的模 $q$ 结果。将其组成向量 $m$，问题变为寻找格点 $As+qk$，使它与 $m$ 的距离最小；差向量就是很小的噪声 $b$。

### Babai 最近平面去噪

构造由 $A$ 的列和 $qI$ 生成的格，使合法无噪向量均属于：

$$
\mathcal L=\{As+qk\mid s\in\mathbb Z^{43},k\in\mathbb Z^{127}\}.
$$

对基执行 LLL，再用 Babai nearest-plane 算法求距目标 $m$ 最近的格点 $v$。验证：

$$
\|m-v\|_\infty
$$

处于噪声应有的小范围内。随后解线性系统 $As=v\pmod q$，把 43 个秘密分量按题目编码规则转回字节。

官方附件的 `A.npy`、`res.txt` 和 `task.sage` 分别给出矩阵、RSA 输出和生成逻辑，三阶段中的维数与模数必须从这些文件读取，不能使用示例常量替代。

## 方法总结

“RSA 模数很大”不能弥补素数生成器的代数结构；同样，“线性方程有噪声”也不能抵抗噪声远小于格间距的 CVP 攻击。复杂密码题应拆成表示层：先恢复模数因子，再剥离 RSA，最后识别剩余样本的格结构，每一层都用代回原式验证。
