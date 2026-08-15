# Esoteric Exponent

## 题目简述

题目只给出 RSA 的 $n$、超大的公钥指数 $e$ 和密文 $c$。这里的大 $e$ 不是异常本身，而是由过小的私钥指数 $d$ 取逆得到的结果。出题约束刻意让 $d$ 超过 Wiener 攻击常见的 $\frac{1}{3}n^{1/4}$ 范围，却仍满足 $d<n^{0.292}$，因此应转向 Boneh–Durfee 小私钥指数攻击。

## 解题过程

RSA 私钥满足

$$
ed-k\varphi(n)=1
$$

其中 $k$ 是未知整数。对平衡素数 $p,q$，有 $p+q=O(\sqrt n)$。令

$$
A=\frac{n+1}{2},\qquad y=-\frac{p+q}{2},
$$

则 $\varphi(n)=2(A+y)$，可以把关系改写为一个模 $e$ 的小根问题：

$$
f(x,y)=1+x(A+y)\equiv0\pmod e,
$$

其中根的量级分别受 $x\approx2k$ 和 $|y|\approx\sqrt n$ 约束。Boneh–Durfee 使用格规约和二元 Coppersmith 小根技术，在 $d<n^\delta$ 且典型上界 $\delta<0.292$ 时恢复该根，随后得到 $d$。

实际求解时，将题目中的 `n`、`e` 交给 Boneh–Durfee 实现，并把小私钥指数界设在 $\delta=0.292$ 以下；得到候选 $d$ 后先验证：

$$
m=c^d\bmod n,qquad m^e\bmod n=c.
$$

解密和验证部分可直接使用：

```python
from Crypto.Util.number import long_to_bytes

n = ...
e = ...
c = ...
d = ...  # Boneh-Durfee 格攻击恢复的候选私钥指数

m = pow(c, d, n)
assert pow(m, e, n) == c
print(long_to_bytes(m))
```

恢复结果为：

```text
shellmates{ev3n_iF_w13NER_fAiLED,_BoNEh_anD_DurFE3_h4vE_yOUR_b4Ck}
```

公开仓库的官方 README 说明了攻击条件和结果，但没有附带具体的 Boneh–Durfee 格构造脚本；网络检索也没有找到可核对的本题专用 solver。因此这里保留了攻击模型、适用界、参数输入和解密验证方式，但不把某个未经本题验证的第三方调参脚本冒充官方实现。

## 方法总结

- 核心技巧：当 RSA 私钥指数满足 $d<n^{0.292}$ 时，使用 Boneh–Durfee 格攻击突破 Wiener 攻击约 $n^{1/4}$ 的边界。
- 识别信号：异常大的 $e$、题目强调“小指数但 Wiener 失败”，或明确给出 $n^{1/4}<d<n^{0.292}$，都指向 Boneh–Durfee。
- 复用要点：格攻击输出的 $d$ 必须通过重新加密验证；不同实现的格维数、`m`、`t` 和 $\delta$ 调参并不通用，不能只凭脚本打印了整数就认定成功。
