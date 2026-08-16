# Chinese Conundrum

## 题目简述

题目分成两层 RSA。内层使用已知的 128 位素数 `P`、`Q` 和一个 32 位素数指数，但发布时从指数二进制串中删除了相邻两位；外层则把内层密文 `enc` 用相同指数 `ec = 71`、十个互素模数分别加密。需要先用广播攻击恢复 `enc`，再补回指数缺失的两位并解开内层 RSA。

## 解题过程

### 用 CRT 恢复外层明文的 71 次幂

十组数据满足：

$$
c_i\equiv enc^{71}\pmod{n_i}
$$

对所有 $(c_i,n_i)$ 使用中国剩余定理，得到：

$$
C\equiv enc^{71}\pmod{M},\qquad M=\prod_i n_i
$$

十个模数的乘积足够大，使真实整数 $enc^{71}<M$，所以 CRT 结果并未发生再次取模；对 `C` 求精确整数 71 次根即可恢复 `enc`。完整脚本会直接从附件 `out.txt` 解析全部参数。

### 枚举被删除的相邻两位

附件给出的 `e` 是删掉两位后的二进制值。原指数只有 32 位，因此枚举插入位置和 `00`、`01`、`10`、`11` 四种组合即可。对每个候选指数，计算私钥指数并检查解密结果：

```python
import ast

import gmpy2
from Crypto.Util.number import long_to_bytes
from math import gcd
from pathlib import Path
from sympy.ntheory.modular import crt

values = {}
for line in Path("out.txt").read_text().splitlines():
    if not line.strip():
        continue
    name, value = line.split("=", 1)
    values[name.strip()] = ast.literal_eval(value.strip())

N = values["N"]
P = values["P"]
Q = values["Q"]
short_e = values["e"]
outer_e = values["ec"]
ciphertexts = values["cipher"]
moduli = values["mod"]
assert N == P * Q and outer_e == 71

combined, _ = crt(moduli, ciphertexts)
enc, exact = gmpy2.iroot(int(combined), outer_e)
assert exact
enc = int(enc)

bits = bin(short_e)[2:]
phi = (P - 1) * (Q - 1)

for pos in range(len(bits) + 1):
    for missing in ("00", "01", "10", "11"):
        candidate_bits = bits[:pos] + missing + bits[pos:]
        candidate_e = int(candidate_bits, 2)
        if candidate_e.bit_length() != 32 or gcd(candidate_e, phi) != 1:
            continue
        d = pow(candidate_e, -1, phi)
        plain = long_to_bytes(pow(enc, d, N))
        if plain.isascii() and all(32 <= byte < 127 for byte in plain):
            print(b"shellmates{" + plain + b"}")
```

最终得到：

```text
shellmates{CRT?l05T_b1t5?_No0o_pR0blem!}
```

## 方法总结

- 核心技巧：先对同指数、不同互素模数的密文做 Håstad 广播攻击，再枚举短指数中缺失的两个相邻 bit。
- 识别信号：相同明文、相同小指数、多个互素模数是广播攻击的典型条件；指数只有少量 bit 遗失时则应直接穷举结构化缺口。
- 复用要点：CRT 后必须验证整数根是精确根；枚举指数时还需检查其位长和与 $\varphi(N)$ 的互素性，避免把无效候选当成私钥参数。
