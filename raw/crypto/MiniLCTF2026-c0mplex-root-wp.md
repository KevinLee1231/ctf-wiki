# c0mplex_root

## 题目简述

公开输出包含 RSA 的 $n,e,\texttt{wrapped\_key\_ct}$，AES-CTR 的 `iv, ct`，以及系数为高斯整数的四次多项式。隐藏根为 $x_0=a+bi$，其中 $0<a,b<2^{128}$，并满足

$$
f(x_0)\equiv0\pmod {p+qi},\qquad n=pq.
$$

题目再以固定宽度编码的 $a,b$ 计算 SHA-256 截断掩码，异或保护 AES key 后用 RSA 加密。故需要同时恢复 RSA 因子和高斯整数小根。原始材料称 RSA 参数存在小私钥指数弱点；这必须用候选 $d$ 的 RSA 关系和所得分解验证，不能仅从“$e$ 接近 $n$”断言。

## 解题过程

### 先验证并利用 RSA 弱私钥

对 $(n,e)$ 运行适用于小 $d$ 的格攻击（原始题解使用 Boneh--Durfee），得到候选 $d$ 后，以

$$
k=ed-1\text{ 的约数关系},\qquad \varphi(n)=\frac{ed-1}{k}
$$

恢复并验证 $p,q$：应同时满足 $pq=n$ 和 $ed\equiv1\pmod{\varphi(n)}$。随后计算

$$
\texttt{wrapped}=\texttt{wrapped\_key\_ct}^{d}\bmod n.
$$

RSA 分解只给出无序因子，后续须分别尝试 $P=p+qi$ 与 $P=q+pi$。

### 把高斯整数同余降为整数同余

对候选 $P=p+qi$ 令 $R=N(P)=p^2+q^2$，并取

$$
t\equiv-pq^{-1}\pmod R.
$$

因为 $t^2\equiv-1\pmod R$ 且 $p+qt\equiv0\pmod R$，映射

$$
u+vi\longmapsto u+vt\pmod R
$$

将 $\mathbb Z[i]/(p+qi)$ 化为同阶的整数商环。把每个系数 $c_j=c_{j,r}+c_{j,i}i$ 映成 $c_{j,r}+c_{j,i}t$，得到

$$
F(X)=\sum_j(c_{j,r}+c_{j,i}t)X^j\pmod R.
$$

于是 $z=a+bt\pmod R$ 是 $F$ 的根。分解 $R$，在每个素数幂上求四次方程根，并用 CRT 组合；任何局部模数无根都可直接淘汰当前 $P$。

### 由 $z$ 恢复 $a,b$ 并解密

整数根只给出

$$
a+bt\equiv z\pmod R.
$$

利用 $a,b<2^{128}$，在由 $(R,0)$ 和 $(-t,1)$ 生成的二维格中，对目标 $(-z,0)$ 做 LLL 约简后的 Babai 近似最近向量。候选必须依次通过范围检查、上式检查，以及原方程 $f(a+bi)\equiv0\pmod P$ 检查。

将 $a,b$ 各按 16 字节大端编码，计算

```python
mask = SHA256(a_bytes + b_bytes).digest()[:16]
wrapped_bytes = wrapped.to_bytes(16, "big")
key = bytes(x ^ y for x, y in zip(wrapped_bytes, mask))
```

其中异或逐字节进行。最后以题目给出的计数器初值做 AES-CTR 解密，并以可读明文/flag 格式验证候选；CTR 模式不涉及 padding。

## 方法总结

- 核心技巧：先由弱 RSA 取得 $p,q$，再用 $i\equiv-pq^{-1}\pmod{p^2+q^2}$ 将高斯整数模根转换为普通整数模根。
- 识别信号：RSA 因子同时被拼入 $p+qi$，并给出小实部、虚部的高斯根时，范数模数是连接两层问题的关键。
- 复用要点：CRT 根、二维 CVP 和原高斯方程必须逐层校验；因子顺序、固定字节宽度与 CTR counter 编码任一处不一致都会得到错误密钥。
