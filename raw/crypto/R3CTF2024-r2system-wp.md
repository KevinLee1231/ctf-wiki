# r2system

## 题目简述

r2system 修复了前两题的直接账户越权，并限制最多注册 10 个账号。注册令牌由一个 7 次多项式状态机生成：

$$
s_{i+1}=f(s_i)=\sum_{j=0}^{7}a_js_i^j\pmod N,
$$

用户名的整数编码记为 $u_i$，服务器秘密值记为 $x$，返回令牌

$$
t_i=x(s_i+u_i)^{-1}\pmod N.
$$

目标仍是预测 Bob 的登录令牌，取得 Bob 的 ECDH 私钥并解密公共频道中的 flag。决定性问题是：10 组已知用户名与令牌足以消去未知多项式系数，恢复 $x$ 和状态机。

## 解题过程

先用满 10 次注册额度，保存每次的用户名整数 $u_i$ 和令牌 $t_i$。由令牌公式可以把每个内部状态写成只含 $x$ 的一次式：

$$
s_i=xt_i^{-1}-u_i\pmod N.
$$

把这些表达式代入连续的状态转移关系 $s_{i+1}=f(s_i)$。未知量原本有 $x$ 和 8 个系数 $a_0,\ldots,a_7$；多项式系数在方程中都是一次的，因此可以通过“相关 nonce 多项式递推”消元，只留下关于 $x$ 的一元多项式。

实现时在 $\operatorname{GF}(N)$ 上建立多项式环。令

$$
k_{i,j}(X)=X(t_i^{-1}-t_j^{-1})-(u_i-u_j),
$$

再使用 Polynonce 攻击中的递归消元构造目标多项式。其核心形式如下：

```python
def k(i, j):
    return X * (pow(t[i], -1, N) - pow(t[j], -1, N)) - (u[i] - u[j])

def dpoly(level, offset):
    if level == 0:
        return k(offset + 1, offset + 2) ** 2 \
             - k(offset + 2, offset + 3) * k(offset, offset + 1)

    left = dpoly(level - 1, offset)
    right = dpoly(level - 1, offset + 1)
    for m in range(1, level + 2):
        left *= k(offset + m, offset + level + 2)
        right *= k(offset, offset + m)
    return left - right
```

对 7 次递推取对应层数并求根，得到若干个 $x$ 候选。对每个候选恢复全部状态：

```python
states = [
    (x * pow(token, -1, N) - username) % N
    for username, token in zip(usernames, tokens)
]
```

然后用连续点 $(s_i,s_{i+1})$ 在 $\operatorname{GF}(N)$ 上做拉格朗日插值。只有插值得到次数恰为 7、且能重放全部观测的候选才保留。计算

$$
s_{10}=f(s_9),\qquad
t_{\text{Bob}}=x(s_{10}+u_{\text{Bob}})^{-1}\pmod N,
$$

即可预测服务器为 `BobCanBeAnyBody` 生成的令牌。

用预测令牌登录 Bob，读取其 ECDH 私钥；再用 Alice 公钥计算共享点，对 `str(shared_point)` 取 MD5 作为 AES-ECB 密钥，解密公共频道密文。若消元产生多个根，就逐个生成 Bob 令牌并尝试登录，服务响应可作为最终判据。

完整的 Sage 消元与交互实现见 [R3CTF 2024 Crypto Writeup](https://tang.cat/2024/06/10/R3CTF-2024-Crypto-Writeup.html)。本文已展开令牌方程、消元输入、候选筛选、状态预测和最终 ECDH 解密链。

## 方法总结

注册次数限制没有阻止攻击，因为“7 次状态多项式 + 1 个服务器秘密值”总计 9 个代数未知量，而 10 组相关输出已经提供足够约束。解决此类题目的关键是先把隐藏状态表示成秘密值 $x$ 的低次式，再利用状态递推消去线性出现的系数。求根后必须通过插值次数和历史样本重放筛选伪解，不能把多项式的第一个根直接当成服务器秘密值。
