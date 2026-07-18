# week1BlackGiveRSA

## 题目简述

题目把 42 字节 flag 每 7 字节分成一组，使用同一组 RSA 参数 $n=1687126110378632809$、$e=10007$ 分别加密，并给出 6 个密文块。目标是分解较小的模数并逐块还原明文。

## 解题过程

模数只有约 61 bit，可以直接分解为

$$
n=1175078221\times1435756429.
$$

于是计算 $\varphi(n)=(p-1)(q-1)$ 和 $d=e^{-1}\bmod\varphi(n)$，再依次解密六个密文块。源码规定每块原文恰好为 7 字节；本题各块最高字节均非零，因此 `long_to_bytes` 后直接拼接即可。

```python
from Crypto.Util.number import inverse, long_to_bytes

p = 1175078221
q = 1435756429
n = p * q
e = 10007
cipher = [
    1150947306854980854,
    243703926267532432,
    1069319314811079682,
    688582941857504686,
    670683629344243145,
    1195068175327355214,
]

d = inverse(e, (p - 1) * (q - 1))
flag = b"".join(long_to_bytes(pow(c, d, n)) for c in cipher)
print(flag.decode())
```

得到：

```text
0xGame{ChuTiRenDeQQShiJiShangJiuShiQDeZhi}
```

## 方法总结

小模数 RSA 的首要检查是能否直接分解 $n$。分块加密时还要按源码恢复原始块顺序和块宽，避免因为整数转换丢失前导零或打乱拼接顺序。
