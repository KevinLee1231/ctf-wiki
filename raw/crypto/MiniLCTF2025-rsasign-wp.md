# rsasign

## 题目简述

题目生成普通 1024-bit RSA：$n=pq$、$e=65537$、$c=m^e\bmod n$。额外泄露

$$
\texttt{gift}=\left\lfloor\frac{(S^2\bmod\varphi(n))}{2^{740}}\right\rfloor,
\qquad S=p+q+a+b+ab,
$$

其中 $a=\operatorname{bytes\_to\_long}(\texttt{b'miniL'})$、$b=\operatorname{bytes\_to\_long}(\texttt{b'mini7'})$，且 $\varphi(n)=n-(p+q)+1$。泄露的是模 $\varphi(n)$ 后的平方高位，足以恢复一个 RSA 因子的高位，再以 Coppersmith 补全低位。

## 解题过程

### 将 gift 写成关于 $p+q$ 的近似方程

设 $r=S^2\bmod\varphi(n)$。由右移操作可知

$$
S^2-k\varphi(n)=\texttt{gift}\cdot2^{740}+u,
\qquad0\le u<2^{740},
$$

其中 $k=\lfloor S^2/\varphi(n)\rfloor$ 是很小的整数。对题目实例，[同届公开复现](https://mi1n9.github.io/2025/05/11/2025miniLCTF/) 得到 $k=4$；稳妥实现应枚举小范围 $k$ 并以后续整除和因子验证筛选，而不是把该数当作所有参数下都成立的常数。

将 $\varphi(n)=n-(p+q)+1$ 代入，$a,b$ 与未知低 740 bit 只影响较低有效位。先保留能决定高位的主项，得到近似关系

$$
(p+q)^2-k\bigl(n-(p+q)+1\bigr)-\texttt{gift}\cdot2^{740}\approx0,
\qquad pq=n.
$$

消去 $q$（例如对这两个多项式取 resultant），在高精度实数域求根，可得到 $p$ 或 $q$ 的高位近似。公开复现中，该近似的高 229 bit 与真实 512-bit 因子一致；这是下一步的输入，不能直接把实数根当作完整因子。

### 以高位泄露补全因子

将近似根向下对齐到 229 个未知低位：

$$
p=p_{\rm high}+x,\qquad |x|<2^{229}.
$$

在 $\mathbb Z/n\mathbb Z$ 上对一元多项式 $f(X)=p_{\rm high}+X$ 使用 Coppersmith small-roots。题目实例可采用 `X = 2^229`、`beta = 0.4`、`epsilon = 0.01`。对每个根候选做确定性验证：

```python
p = p_high + x
assert n % p == 0
q = n // p
phi = (p - 1) * (q - 1)
d = inverse(e, phi)
m = pow(c, d, n)
```

候选不整除 $n$ 时，应换另一个 $k$ 或另一支 resultant 根，而不是放宽 Coppersmith 的界并接受近似因子。

### 解密验证

将 $m$ 转为大端字节并检查 flag 格式。因子分解和 $ed\equiv1\pmod\varphi(n)$ 是密码学层的强验证；若最终字节不是合理明文，应回到高位近似阶段检查 $k$、常数项和移位误差，而非把 RSA 解密解释为编码问题。

## 方法总结

- 核心技巧：模 $\varphi(n)$ 平方结果的高位泄露可以先约束 $p+q$，再转化为单个因子的高位泄露并用 Coppersmith 补全。
- 识别信号：RSA 输出同时给出 $n,c$ 和形如 `pow(p+q+常量, 2, phi) >> t` 的 gift 时，应先写出商 $k$ 与被丢弃低位的精确范围。
- 复用要点：近似方程只负责产生高位候选；最终必须以 $p\mid n$、$q=n/p$、私钥关系和明文格式逐层验证。对本题实例，$k=4$ 是已验证候选而非普遍定理。
