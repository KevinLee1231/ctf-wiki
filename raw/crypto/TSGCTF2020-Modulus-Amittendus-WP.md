# TSGCTF2020 Modulus Amittendus WP

## 题目简述

题目生成普通 2048 位 RSA，却在导出公钥时写错字段：

```ruby
def pubkey
  privkey.to_a[..2].to_h
end

def privkey
  {
    e: @e,
    n: @d,   # 本应是 @n，却泄漏了私钥指数 d
    cf: @cf, # q^{-1} mod p
    p: @p,
    q: @q,
    exp1: @exp1,
    exp2: @exp2,
  }
end
```

附件因而公开了 $e=65537$、私钥指数 $d$ 和 CRT 系数 $c_f=q^{-1}\bmod p$，但没有公开真正的模数 $n=pq$。仅有 $d$ 还不能直接计算 $c^d\bmod n$；需要先由 $ed-1$ 与泄漏的 CRT 系数恢复 $p,q$。

## 解题过程

RSA 满足：

$$
ed-1=k\varphi(n),
\qquad
1\le k<e.
$$

因此枚举 $k=3,4,\ldots,e-1$，只保留能整除 $ed-1$ 的项，就能得到少量候选：

$$
\varphi=(ed-1)/k.
$$

接着利用 $c_fq\equiv1\pmod p$。设 $c_fq=1+tp$，并注意：

$$
\varphi-1=(p-1)(q-1)-1=pq-p-q.
$$

构造：

$$
A=(\varphi-1)c_f+1.
$$

代入上式可得：

$$
A=p\bigl((q-1)c_f-t\bigr),
$$

所以真实候选下必有 $p\mid A$。同时 $(p-1)\mid\varphi$，对与 $p$ 互素的底数 $a$ 有 $a^\varphi\equiv1\pmod p$。于是 $p$ 也是下面两个数的公因子：

$$
A,
\qquad
a^\varphi\bmod A-1.
$$

为排除偶然的额外因子，对多个小底数取连续最大公约数：

```python
from math import gcd

for k in range(3, e):
    if (e * d - 1) % k:
        continue

    phi = (e * d - 1) // k
    A = (phi - 1) * cf + 1

    factor = A
    for base in range(2, 11):
        factor = gcd(factor, pow(base, phi, A) - 1)

    if factor < 100:
        continue

    p = factor
    q = phi // (p - 1) + 1
    n = p * q
    message = pow(ciphertext, d, n)
    flag = message.to_bytes((message.bit_length() + 7) // 8, "big")
    if flag.startswith(b"TSGCTF{"):
        print(flag)
        break
```

求得一个素因子后，另一个由 $\varphi=(p-1)(q-1)$ 直接恢复。最终明文为：

```text
TSGCTF{Okay_this_flag_will_be_quite_long_so_listen_carefully_Happiness_is_our_bodys_default_setting_Please_dont_feel_SAd_in_all_sense_Be_happy!_Anyway_this_challenge_is_simple_rewrite_of_HITCON_CTF_2019_Lost_Modulus_Again_so_Im_very_thankful_to_the_author}
```

## 方法总结

这题的模数虽然缺失，泄漏的 $d$ 仍把 $\varphi(n)$ 限制为 $ed-1$ 的少量候选，CRT 系数又提供了一个含素因子的代数倍数。将两条信息组合后，可以用若干次模幂和最大公约数抽出 $p$。导出密钥时必须使用明确字段而非依赖哈希插入顺序和切片；CRT 参数、私钥指数和中间指数都属于私钥材料，任何一个泄漏都可能与其他公开关系组合成完整分解。
