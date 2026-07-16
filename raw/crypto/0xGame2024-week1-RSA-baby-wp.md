# RSA-baby

## 题目简述

题目完整给出 RSA 公钥、私钥指数和密文：

```text
N = 547938466798424179
e = 80644065229241095
d = 488474228706714247
c = 344136655393256706
```

源码随机选择整数 $m$，计算 $c\equiv m^e\pmod N$，并把 `MD5(str(m))` 放入 flag。由于私钥指数 $d$ 已经给出，本题只考查标准 RSA 解密和题目规定的结果格式。

## 解题过程

RSA 密钥满足：

$ed\equiv1\pmod{\varphi(N)}$。

因此：

$c^d\equiv m^{ed}\equiv m\pmod N$。

按源码对解密出的整数取十进制字符串的 MD5：

```python
from hashlib import md5

N = 547938466798424179
d = 488474228706714247
c = 344136655393256706

m = pow(c, d, N)
digest = md5(str(m).encode()).hexdigest()
print(f"0xGame{{{digest}}}")
```

输出为：

```text
0xGame{6e5719c54cdde25ce7124e280803f938}
```

## 方法总结

- 核心技巧：使用已给出的私钥指数执行模幂解密，再严格按源码规定计算 MD5。
- 识别信号：RSA 参数中同时出现 `(N,e)`、`(N,d)` 和密文时，不需要分解 $N$。
- 复用要点：先核对 flag 的派生方式；本题哈希的是整数十进制字符串，不是 `long_to_bytes(m)`，两者结果不同。
