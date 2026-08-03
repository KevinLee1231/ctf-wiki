# UIUCTF 2023 crack_the_safe Writeup

## 题目简述

题目把 16 字节 AES 密钥解释为整数 $k$，并公开素数 $p$、底数 7 和目标值 $h$，满足：

$$
7^k\equiv h\pmod p.
$$

其中：

```text
p = 4170887899225220949299992515778389605737976266979828742347
h = 0x49545b7d5204bd639e299bc265ca987fb4b949c461b33759
```

求出离散对数 $k$ 后，将其编码为 16 字节 AES-128 密钥，即可解密给出的 ECB 密文。

## 解题过程

首先分解群阶 $p-1$：

```text
p - 1 = 2 * 19 * 151 * 577 * 67061
          * 18279232319 * 111543376699
          * 9213409941746658353293481
```

各因子互异，且 7 在模 $p$ 乘法群中具有完整的 $p-1$ 阶。因此可以使用 Pohlig-Hellman：分别在每个素因子阶的子群中求 $k$ 的剩余，再通过中国剩余定理合并。

除最大因子外，其余子群可直接用 baby-step giant-step 求解。最大因子：

```text
ell = 9213409941746658353293481
```

约为 83 位，通用 BSGS 的内存和时间成本过高。官方解法使用 CADO-NFS 的离散对数模式，分别计算目标 $h$ 和底数 7 相对于同一内部基准的虚拟对数：

```text
log(h) = 2215765705042274080663116 mod ell
log(7) = 6424341129540508417798214 mod ell
```

因为 $\log(h)=k\log(7)\pmod\ell$，所以：

$$
k\equiv\log(h)\cdot\log(7)^{-1}
 \equiv741784031885807265615861\pmod\ell.
$$

把这项预计算结果与其他小子群的离散对数一起交给 Sage：

```python
from sage.all import Mod, crt, factor
from sage.groups.generic import bsgs

p = 4170887899225220949299992515778389605737976266979828742347
g = Mod(7, p)
h = Mod(0x49545b7d5204bd639e299bc265ca987fb4b949c461b33759, p)

large_factor = 9213409941746658353293481
large_residue = 741784031885807265615861

residues = []
moduli = []
for prime_factor, exponent in factor(p - 1):
    subgroup_order = int(prime_factor**exponent)
    projected_g = g ** ((p - 1) // subgroup_order)
    projected_h = h ** ((p - 1) // subgroup_order)

    if int(prime_factor) == large_factor:
        residue = large_residue
    else:
        residue = bsgs(projected_g, projected_h, (0, subgroup_order - 1))

    residues.append(int(residue))
    moduli.append(subgroup_order)

k = int(crt(residues, moduli))
assert pow(7, k, p) == int(h)
print(k)
```

合并后得到：

```text
k = 201920744490721838622302286278878924260
key = 97e885548276adba85f3cc1cd6c58de4
```

最后解密：

```python
from Crypto.Cipher import AES

ct = bytes.fromhex(
    "ae7d2e82a804a5a2dcbc5d5622c94b3e"
    "14f8c5a752a51326e42cda6d8efa4696"
)
key = k.to_bytes(16, "big")
print(AES.new(key, AES.MODE_ECB).decrypt(ct).decode())
```

输出为：

```text
uiuctf{Dl0g_w/__UnS4F3__pR1Me5_}
```

## 方法总结

题目把 AES 密钥直接暴露为有限域离散对数，并选择了群阶可完全分解的素数模数。Pohlig-Hellman 会把原问题拆成若干子群问题，复杂度由最大素因子主导；较小因子用 BSGS，最大的 83 位因子用 NFS-DLP，最后以 CRT 还原指数。生成离散对数参数时，仅保证模数为素数并不够，还必须确保所用子群具有足够大的素数阶。
