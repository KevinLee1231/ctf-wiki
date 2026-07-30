# L3akCTF 2024 Paillien Tourist Writeup

## 题目简述

题目实现 Paillier 加法同态加密，公开两个普通消息的密文、一个被修改的 flag 密文，以及公私钥。服务端先计算：

$$
C_\Delta=E(m_2-m_1),
\qquad
C_{\text{modified}}=E(\text{flag})\cdot C_\Delta.
$$

Paillier 在模 $n^2$ 的密文乘法对应模 $n$ 的明文加法，因此不需要分别解出两个普通消息；只需在密文域撤销差值，再用给出的私钥解密。

## 解题过程

对 Paillier 密文有：

$$
E(a)E(b)=E(a+b),
\qquad
E(a)E(b)^{-1}=E(a-b).
$$

题目给出的两个密文分别是 $C_1=E(m_1)$ 与 $C_2=E(m_2)$。先构造差值密文：

$$
C_\Delta=C_2C_1^{-1}\bmod n^2.
$$

修改后的 flag 密文加上了同一个差值，因此：

$$
C_{\text{flag}}
=C_{\text{modified}}C_\Delta^{-1}\bmod n^2.
$$

题目使用 $g=n+1$，并直接公开私钥参数 $\lambda$ 与 $\mu$。定义：

$$
L(u)=\frac{u-1}{n},
$$

即可按标准 Paillier 公式恢复明文：

$$
m=L(C_{\text{flag}}^\lambda\bmod n^2)\mu\bmod n.
$$

对应的精简求解代码为：

```python
n2 = n * n
c_delta = c2 * pow(c1, -1, n2) % n2
c_flag = c_modified * pow(c_delta, -1, n2) % n2

L = lambda value: (value - 1) // n
m = L(pow(c_flag, lam, n2)) * mu % n
print(long_to_bytes(m))
```

官方 solver 使用题目输出中的完整参数执行上述运算，并以解出的 `L3AK{...}` 字节串作为验证。仓库没有另外保存真实 flag 明文，因此不应把生成脚本里的 `L3AK{FAKE_FLAG_FAKE_FLAG}` 当成比赛答案。

## 方法总结

- 核心技巧：使用 Paillier 的加法同态性质，在密文域消去人为叠加的消息差值。
- 识别信号：看到 $g=n+1$、模 $n^2$ 运算以及密文乘除，应把乘法解释为明文加减，而不是普通 RSA 运算。
- 复用要点：先写清每个密文对应的明文表达式，再在密文域做代数消元；构建脚本中的假 flag 只能说明格式，不能作为真实验证结果。
