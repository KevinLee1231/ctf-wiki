# RSA-easy

## 题目简述

题目只给出 RSA 公钥与密文：

```text
N = 689802261604270193
e = 620245111658678815
c = 289281498571087475
```

源码使用两个约 30 bit 的素数生成 $N=pq$。因此模数总长只有约 60 bit，可以在本地快速分解；得到 $p,q$ 后即可计算 $\varphi(N)$ 和私钥指数。flag 仍定义为 `MD5(str(m))`。

## 解题过程

分解模数得到：

```text
p = 823642439
q = 837502087
```

计算：

$\varphi(N)=(p-1)(q-1)$，

$d\equiv e^{-1}\pmod{\varphi(N)}$，

$m\equiv c^d\pmod N$。

完整脚本如下，`factorint()` 会直接处理这个小模数，不依赖在线分解数据库：

```python
from hashlib import md5
from sympy import factorint

N = 689802261604270193
e = 620245111658678815
c = 289281498571087475

p, q = factorint(N).keys()
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
m = pow(c, d, N)

digest = md5(str(m).encode()).hexdigest()
print(f"0xGame{{{digest}}}")
```

输出为：

```text
0xGame{5aa4603855d01ffdc5dcf92e0e604f31}
```

## 方法总结

- 核心技巧：分解过小的 RSA 模数，恢复 $\varphi(N)$ 与私钥指数。
- 识别信号：$N$ 只有几十 bit，远低于实际 RSA 安全规模，说明大整数分解不是障碍。
- 复用要点：RSA 安全性依赖模数难以分解；得到 $p,q$ 后，后续只剩标准逆元和模幂运算，同时要核对题目对明文的二次编码或哈希方式。
