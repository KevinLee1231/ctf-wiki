# Groups

## 题目简述

服务要求先提交一个至少 512 位的合数 $c$。它用 50 轮随机底数检查 $a^{c-1}\equiv1\pmod c$，通过后随机生成互素底数 $a$ 和指数 $k$，给出 $b=a^k\bmod c$，要求回交任意满足同一等式的指数。关键并非直接攻击一个未知大素数群，而是主动选择结构已知且群阶光滑的 Carmichael 数。

## 解题过程

采用 Chernick 构造：若

$$
p_1=6n+1,\qquad p_2=12n+1,\qquad p_3=18n+1
$$

均为素数，则 $c=p_1p_2p_3$ 是 Carmichael 数。选择本身由许多小素数组成的光滑整数 $n$，既能让 $c$ 通过题目的 Fermat 伪素数检查，又能让

$$
\varphi(c)=(6n)(12n)(18n)
$$

完全可分解为小素数幂。官方解使用下面这组已验证参数：

```text
n = 5398029043857557866994901330452977650335980815087189799906
c = 203849970210091453714531207022431893145684152100740176951060094136585680367284801433717312643035947740122153055003391810969143398155610358038226586501787583449037375466575858809
```

收到 $a,b$ 后，需要解 $a^x=b\pmod c$。不知道 $a$ 的精确阶并不妨碍求解，因为它的阶一定整除 $\varphi(c)$。对 $\varphi(c)=\prod q_i^{e_i}$ 的每个素数幂，把等式同时提升到 $\varphi(c)/q_i^{e_i}$ 次幂：

$$
\left(a^{\varphi(c)/q_i^{e_i}}\right)^x
=b^{\varphi(c)/q_i^{e_i}}\pmod c.
$$

每个小子群中的离散对数可用 BSGS 求得，再用 CRT 合并各个同余。Sage 的通用离散对数入口可能先尝试计算大合数模乘法群的精确阶，因此直接调用 `bsgs` 更稳妥：

```python
from sage.all import CRT_list, Mod, factor
from sage.groups.generic import bsgs

phi = (6 * n) * (12 * n) * (18 * n)
a = Mod(a_from_server, c)
b = Mod(b_from_server, c)

residues = []
moduli = []
for prime, exponent in factor(phi):
    modulus = prime ** exponent
    base = a ** (phi // modulus)
    target = b ** (phi // modulus)
    residues.append(bsgs(base, target, (0, modulus)))
    moduli.append(modulus)

x = int(CRT_list(residues, moduli))
assert pow(int(a), x, c) == int(b)
```

服务只验证模幂结果而不要求恢复它最初抽取的那个整数，因此 CRT 得到的等价指数可以直接提交。成功后得到：

```text
uiuctf{c4rm1ch43l_7adb8e2f019bb4e0e8cd54e92bb6e3893}
```

## 方法总结

- 核心技巧：利用题目允许自选模数，把困难的未知群离散对数改造成已知光滑群阶上的 Pohlig--Hellman。
- Carmichael 数只保证所有互素底数通过 Fermat 型测试；这里仍需确保 $c$ 是合数、超过 512 位，且三个线性因子均为素数。
- 不必求出底数的真实阶。用其已知倍数 $\varphi(c)$ 分解子问题，并在提交前验证 $a^x\equiv b\pmod c$，即可处理非生成元带来的多解。
