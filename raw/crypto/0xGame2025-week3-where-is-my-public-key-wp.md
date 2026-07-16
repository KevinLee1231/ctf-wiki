# where is my public key

## 题目简述

服务使用 RSA，但菜单 1 只给出密文 $c$ 和两个素因子 $p,q$，故意隐藏随机生成的 300 位公钥指数 $e$；菜单 2 则提供任意整数的加密 oracle。`p`、`q` 的生成方式使 $p-1$ 与 $q-1$ 完全由小素数组成，因此可通过一次选择明文查询，把恢复 $e$ 转化为光滑阶群上的离散对数。

## 解题过程

先从菜单 1 取得 $c,p,q$，计算：

$$
n=pq,\qquad \varphi(n)=(p-1)(q-1).
$$

源码的 `getSmoothPrime()` 先连乘小素数，再检查“乘积加一”是否为素数，所以 $\varphi(n)$ 的分解已知且非常光滑。向加密 oracle 提交 $m=2$，得到：

$$
c_2\equiv 2^e\pmod n.
$$

于是 $e$ 就是以 $2$ 为底、以 $c_2$ 为目标的离散对数。Pohlig–Hellman 在光滑阶上可以高效完成该计算。恢复 $e$ 后按正常 RSA 流程求私钥指数并解密：

```sage
from sage.all import Zmod, discrete_log, inverse_mod
from Crypto.Util.number import long_to_bytes

# 由菜单 1 取得
c = ...
p = ...
q = ...

n = p * q
phi = (p - 1) * (q - 1)

# 向菜单 2 提交 2 后取得
c2 = ...

ring = Zmod(n)
e = discrete_log(ring(c2), ring(2), ord=phi)
assert pow(2, int(e), n) == c2

d = inverse_mod(e, phi)
m = pow(c, int(d), n)
flag = long_to_bytes(m).rstrip(b'\x00')
print(flag)
```

末尾的零字节来自源码把 `flag` 右侧补齐到 50 字节，解密后应删除。严格说，离散对数先确定的是 $e$ 对底数 $2$ 的阶取模后的值；本题生成的阶足够大，且源码限定 $e$ 为 300 位。若某次实例出现多个候选，可再查询另一个与 $n$ 互素的底数并联立验证。

## 方法总结

- 本题并不是缺少整个 RSA 公钥，而是只缺少指数 $e$；反而直接泄露了本应保密的 $p,q$。
- 选择明文 oracle 给出了 $m^e\bmod n$，可把未知指数变成离散对数问题。
- 看到 $p-1$、$q-1$ 由小素数组成时，应优先考虑 Pohlig–Hellman，而不是暴力枚举 300 位指数。
