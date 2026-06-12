# Ez_RSA

## 题目简述

RSA 题。hint 暴露了可从两端同时约束 `p`、`q` 的比特关系，利用 DFS 从高位和低位同步枚举，并用乘积区间与低位模约束剪枝，恢复因子后常规解密。

## 解题过程

dfs+剪枝

```python
from Crypto.Util.number import *
nbits = 512
n =
e = 65537
c =
hint =
bhint = bin(hint)[2:].zfill(nbits)
def dfs(p, pp, q, qq):
    if len(p) == nbits // 2:
        ans1, ans2 = int(p + pp, 2), int(q + qq, 2)
        if ans1 * ans2 == n:
            return (ans1, ans2)
        else:
            return None
    nn = len(p)
    bit1 = bhint[nn]
    bit2 = bhint[-(nn + 1)]
    for i in range(2):
        for j in range(2):
            ii = i ^ int(bit1)
            jj = j ^ int(bit2)
            tmp_p = p + str(i)
            tmp_pp = str(j) + pp
            tmp_q = q + str(jj)
            tmp_qq = str(ii) + qq
            if int(tmp_p + "1" * (nbits - 2 - 2 * nn) + tmp_pp, 2) * int(tmp_q
+ "1" * (nbits - 2 - 2 * nn) + tmp_qq, 2) >= n and\
               int(tmp_p + "0" * (nbits - 2 - 2 * nn) + tmp_pp, 2) * int(tmp_q
+ "0" * (nbits - 2 - 2 * nn) + tmp_qq, 2) <= n and\
               int(tmp_pp, 2) * int(tmp_qq, 2) % (1 << (nn + 1)) == n % (1 <<
(nn + 1)):
                res = dfs(tmp_p, tmp_pp, tmp_q, tmp_qq)
                if res:
                    return res
p, q = dfs("1", "1", "1", "1")
assert p * q == n
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, n)
flag = long_to_bytes(m)
print(flag)
# nctf{4lg0r1thm1c_1s_a1s0_1mp0rt4nt!}
```

## 方法总结

- 核心技巧：基于 hint 的 RSA 因子比特 DFS 与区间剪枝。
- 识别信号：hint 可拆成与 p/q 高低位相关的二进制串时，优先考虑双端 DFS。
- 复用要点：枚举时同时维护乘积上下界和低位乘积模约束，可以显著降低搜索空间。
