# Bad RSA

## 题目简述

题目给出素数模数 `n`、指数 `e = 32` 和密文 `c`，满足 $c \equiv m^{32} \pmod n$。这里不能按常规 RSA 直接计算 $e^{-1} \bmod \varphi(n)$：因为 $e=2^5$ 与 $n-1$ 不互素，逆元通常不存在。真正的突破口是模数本身为素数，可以在有限域中逐层求模平方根。

## 解题过程

### 把 32 次幂拆成 5 次平方

由于 $32=2^5$，加密关系可写成：

$$
c \equiv ((((m^2)^2)^2)^2)^2 \pmod n
$$

因此从 `c` 开始连续求 5 次模平方根，就能枚举所有可能的 `m`。每次若 $r^2\equiv x\pmod n$，则 $-r\bmod n$ 也是根，所以不能只沿一条分支向下求解。

下面的代码直接读取附件 `Bad_RSA.txt`，并使用可返回全部根的 `sqrt_mod`：

```python
from Crypto.Util.number import long_to_bytes
from pathlib import Path
from sympy.ntheory.residue_ntheory import sqrt_mod

values = {}
for line in Path("Bad_RSA.txt").read_text().splitlines():
    name, value = line.split("=", 1)
    values[name.strip()] = int(value)

c = values["c"]
n = values["n"]
assert values["e"] == 32

candidates = {c}
for _ in range(5):
    next_candidates = set()
    for value in candidates:
        roots = sqrt_mod(value, n, all_roots=True)
        next_candidates.update(int(root) for root in roots)
    candidates = next_candidates

for value in candidates:
    plaintext = long_to_bytes(value)
    if b"shellmates{" in plaintext:
        print(plaintext)
```

对当前仓库 `challenge/Bad_RSA.txt` 中的参数运行后，在至多 $2^5=32$ 个候选中找到：

```text
shellmates{so_modular_square_root_is_a_thing}
```

仓库的 solution README 和 `solution.py` 使用了另一组 `c`、`n`，对应的 flag 是 `shellmates{S0_M0dul4r_Squ4r3_R00t_1s_4_Th1ng}`。两组数据的攻击方法相同，但不能把 solution 中旧实例的结果冒充为当前附件的解密结果；本 WP 以可复现的 `challenge/Bad_RSA.txt` 为准。

## 方法总结

- 核心技巧：识别 $e=2^k$，将模幂逆运算转化为连续 $k$ 次模平方根。
- 识别信号：RSA 模数异常地直接给成素数，且公钥指数与 $n-1$ 不互素时，应检查模根枚举而不是强求私钥指数。
- 复用要点：每层平方根通常有正负两个分支，必须保留完整候选树，再利用 flag 格式或可打印性筛选。
