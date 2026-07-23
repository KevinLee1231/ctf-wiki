# Eeveelutions

## 题目简述

生成器使用同一个 2048 位 RSA 模数 $n$ 和小指数 $e=3$ 加密两条相关消息。原始 flag 中的字符串 `eevee` 会分别替换成两个不同的宝可梦进化名，末尾再各加 12 个随机字母数字字符：

```python
flag1 = flag.replace("eevee", evolution_1) + pad1
flag2 = flag.replace("eevee", evolution_2) + pad2

c1 = pow(bytes_to_long(flag1.encode()), 3, n)
c2 = pow(bytes_to_long(flag2.encode()), 3, n)
```

两条明文的大部分内容相同，差异只来自已知候选进化名和很短的随机后缀。这正是 RSA 小指数相关消息攻击的适用条件。

## 解题过程

### 分离已知差异和短未知差异

设两条整数明文为 $m_1,m_2$，其差为

$$
m_2-m_1=\Delta_{\text{pad}}+2^i\Delta_{\text{name}}.
$$

$\Delta_{\text{name}}$ 可以从候选集合
`umbreon`、`sylveon`、`jolteon`、`flareon`、`glaceon`、`leafeon`
两两计算；$i$ 是替换位置对应的字节偏移，可按 8 位边界枚举。真正未知的
$\Delta_{\text{pad}}$ 只有约 12 字节，因此相对 2048 位模数很小。

### Coppersmith 短填充攻击

在 $\mathbb Z_n[x,y]$ 上写出

$$
\begin{aligned}
f_1(x)&=x^3-c_1,\\
f_2(x,y)&=(x+y+2^i\Delta_{\text{name}})^3-c_2.
\end{aligned}
$$

对 $x$ 求结果式，得到只含短差值 $y$ 的多项式，再对其使用
Coppersmith `small_roots()`。官方脚本对所有进化名差值和合理字节偏移进行枚举，找到满足界限的小根
$\Delta_{\text{pad}}$。

### Franklin-Reiter 恢复明文

得到完整差值 $\Delta=m_2-m_1$ 后，两条同模同指数 RSA 多项式为

$$
g_1(X)=X^3-c_1,\qquad
g_2(X)=(X+\Delta)^3-c_2.
$$

它们在 $\mathbb Z_n[X]$ 上共享线性因子 $X-m_1$。使用多项式欧几里得算法计算
$\gcd(g_1,g_2)$，即可从该一次因子读出 $m_1$：

```python
R.<X> = Zmod(n)[]
g1 = X^3 - c1
g2 = (X + delta)^3 - c2
common = polynomial_gcd(g1, g2)
message = int(n - common[0])
print(long_to_bytes(message))
```

恢复出的文本包含某个实际进化名和 12 字节随机尾缀；按照题目生成逻辑将进化名还原为
`eevee` 并去掉随机后缀，得到：

```text
UMDCTF{y0Ur_-__eevee_h4s_eeV0Lv3d_1nt0_CRypT30N}
```

## 方法总结

- 核心技巧：先用 Coppersmith 短填充攻击恢复两条明文的小差值，再用 Franklin-Reiter 相关消息攻击求出原文。
- 识别信号：同一 RSA 模数、$e=3$、两条高度相似明文和很短的独立后缀同时出现时，应检查相关消息而非尝试分解 $n$。
- 复用要点：结构化差异要拆成“可枚举的已知部分”和“小范围未知部分”；字节序与差异所处的 $2^i$ 位移必须和整数编码一致。
