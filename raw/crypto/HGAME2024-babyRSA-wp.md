# babyRSA

## 题目简述

题目使用非标准模数 $n=p^4q$，密文为 $c\equiv m^e\pmod n$，但没有直接给出指数 $e$。额外泄露值满足：

$$
\text{gift}\equiv\left(e+114514+p^k\right)^{65537}\pmod p,
$$

其中 $k$ 为正素数。需要先在模 $p$ 下消去 $p^k$ 并恢复 $e$，再处理 $e\mid\varphi(n)$ 导致普通 RSA 私钥指数不存在的问题。

## 解题过程

### 从 gift 恢复公开指数

因为 $p^k\equiv0\pmod p$，所以：

$$
\text{gift}\equiv(e+114514)^{65537}\pmod p.
$$

又因为 $\gcd(65537,p-1)=1$，可计算

$$
d_1\equiv65537^{-1}\pmod{p-1},
$$

并在模 $p$ 的乘法群中取回唯一的 $65537$ 次根：

$$
e\equiv\text{gift}^{d_1}-114514\pmod p.
$$

代入题目参数得到 $e=73561$。这一步不需要爆破。

### 在 $\mathbb Z_n$ 中枚举高次根

对于 $n=p^4q$：

$$
\varphi(n)=p^3(p-1)(q-1).
$$

计算可知 $73561\mid\varphi(n)$，所以 $e$ 在模 $\varphi(n)$ 下没有逆元，不能使用普通 RSA 的 $d=e^{-1}\bmod\varphi(n)$。题目需要直接求解：

$$
x^{73561}\equiv c\pmod n.
$$

SageMath 的 `nth_root(..., all=True)` 可以在模环中枚举全部高次根；底层思路可视为对各素数幂分量求根后再组合，常见实现会使用 Adleman–Manders–Miller 一类有限域开根算法。最后用 `hgame{` 前缀筛选真实明文。

```python
from Crypto.Util.number import inverse, long_to_bytes

p = 14213355454944773291
q = 61843562051620700386348551175371930486064978441159200765618339743764001033297
c = 105002138722466946495936638656038214000043475751639025085255113965088749272461906892586616250264922348192496597986452786281151156436229574065193965422841
gift = 9751789326354522940

d1 = inverse(65537, p - 1)
e = (pow(gift, d1, p) - 114514) % p
assert e == 73561

n = p^4 * q
phi = p^3 * (p - 1) * (q - 1)
assert phi % e == 0

ring = Zmod(n)
for root in ring(c).nth_root(e, all=True):
    plaintext = long_to_bytes(int(root))
    if plaintext.startswith(b"hgame{"):
        print(plaintext)
        break
```

输出为：

```text
hgame{Ad1eman_Mand3r_Mi11er_M3th0d}
```

注意这段代码必须在 SageMath 中运行；其中 `^` 表示乘方。若改写成普通 Python，则必须使用 `**`，否则 `^` 会变成按位异或。官方 PDF 没有展示最终输出，flag 由[参赛者题解](https://www.cnblogs.com/mumuhhh/p/18012237)补充核验。

## 方法总结

- 看到底数中含 $p^k$ 且整体对 $p$ 取模，应先用 $p^k\equiv0\pmod p$ 化简。
- 指数取逆发生在乘法群的阶 $p-1$ 上，而不是模 $p$ 直接求逆；恢复的是 $e+114514$ 的群元素。
- 恢复 $e$ 后必须检查 $\gcd(e,\varphi(n))$。若不互素，标准 RSA 解密公式失效，明文可能对应多个模根。
- 枚举高次根后要结合 flag 格式、重新加密验证等条件筛选，不能默认任意一个根都是真实明文。
