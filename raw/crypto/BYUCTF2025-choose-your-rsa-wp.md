# Choose Your RSA

## 题目简述

服务生成 160 字节随机量 $m$，取其前 16 字节作为 AES-ECB 密钥加密 flag；随后分别生成约 2048、3072、4096 位的三个 RSA 模数，并允许选手提交三个严格递增且大于 1 的指数 $e_0,e_1,e_2$，服务返回 $c_i=m^{e_i}\bmod n_i$。

可控指数让三个不同幂次被提升为同一个低次幂，再通过中国剩余定理拼接。

## 解题过程

选择指数 `2 3 6`。于是：

$$
c_0^3\equiv m^6\pmod{n_0},\qquad
c_1^2\equiv m^6\pmod{n_1},\qquad
c_2\equiv m^6\pmod{n_2}.
$$

对这三个余数执行 CRT，得到 $m^6$ 在 $N=n_0n_1n_2$ 下的唯一代表。$m$ 只有 1280 位，所以 $m^6$ 最多约 7680 位；三个模数总长度约为 9216 位，因此 $m^6<N$，CRT 结果就是整数域中的真实 $m^6$，不存在模回绕。

```python
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes
from Crypto.Util.Padding import unpad
from sympy import integer_nthroot
from sympy.ntheory.modular import crt

residues = [pow(c0, 3, n0), pow(c1, 2, n1), c2]
m6 = int(crt([n0, n1, n2], residues)[0])
m, exact = integer_nthroot(m6, 6)
assert exact

key = long_to_bytes(m)[:16]
flag = unpad(AES.new(key, AES.MODE_ECB).decrypt(bytes.fromhex(enc)), 16)
print(flag)
```

这里的 `integer_nthroot` 应使用返回“根与是否精确”的整数实现，避免浮点数精度丢失。解密结果为：

```text
byuctf{Chin3s3_rema1nd3r_th30r3m_is_Sup3r_H3lpful}
```

## 方法总结

- 核心技巧：通过选择 $2,3,6$ 把异指数 RSA 输出统一为 $m^6$，再做 CRT 与精确整数开根。
- 识别信号：同一明文在互素模数下加密、指数可控且明文相对模数乘积足够小时，应检查广播攻击的广义形式。
- 复用要点：CRT 后能否直接开根取决于明确的位数上界；必须先证明 $m^k<\prod n_i$。
