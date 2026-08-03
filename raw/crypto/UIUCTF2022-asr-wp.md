# asr

## 题目简述

题目实现了一组没有公开模数 $n$ 的 RSA 参数，只给出公钥指数 $e=65537$、私钥指数 $d$ 和密文 $c$。两个素数并非普通随机素数，而是先各取 8 个约 64 位素数相乘，再依次乘入少量小素数，直到“乘积加一”为素数：

```python
def gen_prime(bits, lim=7, sz=64):
    while True:
        base = prod(getPrime(sz) for _ in range(bits // sz))
        for i in range(lim):
            if isPrime(base + 1):
                return base + 1
            base *= small_primes[i]
```

因此 $p-1$ 与 $q-1$ 都由一批约 64 位素因子和很小的修正因子组成。题目的关键不是直接猜测 $n$，而是利用泄露的 $d$ 恢复结构化的 $\varphi(n)$，进而重建 $p$、$q$ 和 $n$。

## 解题过程

RSA 私钥指数满足

$$
ed \equiv 1 \pmod{\varphi(n)},
$$

所以存在整数 $k$ 使得

$$
ed-1=k\varphi(n)=k(p-1)(q-1).
$$

先在 SageMath 中分解 $ed-1$。生成器放入 $p-1$、$q-1$ 的 64 位素数会直接出现在这个分解中，而 $k$ 和生成器追加的修正量只会贡献较小的因子。官方 solver 因而提取所有大于 $2^{60}$ 的素因子，再把它们平均分成两组，分别作为 $p-1$ 与 $q-1$ 的主体。

生成器依次乘入小素数，所以需要尝试的修正量是小素数前缀积，例如 $1,2,6,30,210,\ldots$。对每种划分和修正量组合，检查

$$
p=P\cdot h_p+1,\qquad q=Q\cdot h_q+1
$$

是否同时为素数。下面是与官方脚本等价、但避免在循环中原地累乘造成歧义的核心逻辑：

```python
from itertools import combinations
from math import prod
from Crypto.Util.number import isPrime, long_to_bytes
from sage.all import factor

large = [int(r) for r, _ in factor(e * d - 1) if r > 2**60]
prefix = [1, 2, 6, 30, 210, 2310, 30030]

for left_idx in combinations(range(len(large)), len(large) // 2):
    left_idx = set(left_idx)
    P = prod(large[i] for i in left_idx)
    Q = prod(large[i] for i in range(len(large)) if i not in left_idx)

    for hp in prefix:
        p = P * hp + 1
        if not isPrime(p):
            continue
        for hq in prefix:
            q = Q * hq + 1
            if not isPrime(q):
                continue
            n = p * q
            plaintext = long_to_bytes(pow(ct, d, n))
            if plaintext.startswith(b"uiuctf{"):
                print(plaintext)
```

找到正确的两个素数后，使用恢复出的 $n=pq$ 直接计算 $m=c^d\bmod n$。官方参数得到：

```text
uiuctf{bru4e_f0rc3_1s_FUn_fuN_Fun_f0r_The_whOLe_F4miLY!}
```

## 方法总结

- 核心技巧：利用 $ed-1$ 是 $\varphi(n)$ 的倍数，并结合 $p-1$、$q-1$ 的平滑结构恢复缺失的 RSA 模数。
- 识别信号：题目公开 $e,d,c$ 却隐藏 $n$，同时素数生成器让 $p-1$ 和 $q-1$ 由可分解的小块构成。
- 复用要点：先分解 $ed-1$ 并区分“大结构因子”和“小修正因子”；划分因子后必须用素性和明文格式双重验证候选，不能只凭乘积规模判断。
