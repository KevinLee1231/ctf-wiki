# u-235

## 题目简述

题目实现 ElGamal 风格签名，并把 flag 整数 $s$ 隐藏进每次签名的 nonce：

$$
k=s+t m.
$$

公开素数满足 $p-1=mq$，其中 $m$ 是约 256 bit 的光滑整数，$q$ 是大因子。签名公开 $r=g^k\bmod p$，目标是从 $r$ 中恢复 $k\bmod m$，也就是 flag 整数 $s$。

## 解题过程

把公开值提升到 $q=(p-1)/m$ 次方：

$$
r^q=(g^k)^q=(g^q)^k\pmod p.
$$

令 $h=g^q$。因为

$$
h^m=g^{qm}=g^{p-1}=1\pmod p,
$$

$h$ 位于阶整除 $m$ 的子群中。因此指数只保留模 $m$ 的信息：

$$
r^q=h^{s+tm}=h^s\pmod p.
$$

现在只需求子群中的离散对数

$$
s=\log_h(r^q)\pmod m.
$$

$m$ 被刻意构造成光滑数，所以 SageMath 的 `discrete_log` 会使用 Pohlig-Hellman 一类方法快速完成，而无需在整个 $p-1$ 阶群上求解：

```python
# SageMath
from Crypto.Util.number import long_to_bytes

p = Integer(PUBLIC_P)
g = Integer(PUBLIC_G)
m = Integer(SMOOTH_M)
r = Integer(SIGNATURE_R)

q = (p - 1) // m
base = Mod(g, p) ** q
value = Mod(r, p) ** q
secret = discrete_log(value, base, ord=m)
print(long_to_bytes(int(secret)))
```

运行后恢复：

```text
tjctf{hay_gato_encerrado28f23}
```

## 方法总结

- nonce 只要以“秘密加公开周期倍数”的形式构造，就可能在某个子群投影下直接暴露秘密余数。
- 指数乘以 $q=(p-1)/m$ 相当于把元素投影到 $m$ 阶子群；光滑的 $m$ 让离散对数从困难问题变成可分解的小阶问题。
- ElGamal 签名公式本身不是攻击重点，真正的漏洞是 nonce 的代数结构。分析签名题时应优先审计随机数如何生成，而不是只盯着验证等式。
