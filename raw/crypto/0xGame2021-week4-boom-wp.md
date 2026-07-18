# week4boom

## 题目简述

题目使用 32 bit 线性同余生成器，但只输出每个内部状态的高 16 位。已知连续两个截断输出 `state1`、`state2`，密文则逐字节异或后续输出模 10 的结果。

## 解题过程

LCG 内部递推为

$$
s_{i+1}=(as_i+b)\bmod m,
$$

而泄露值是 $s_i\gg16$。第一个完整状态可写成

$$
s_1=(14216\ll16)+r,qquad0\le r<2^{16}.
$$

只需枚举低 16 位 $r$，计算 $s_2$ 并检查 $s_2\gg16=2162$。这将搜索空间从 $2^{32}$ 降为 $2^{16}$，而且本题数据只有一个候选：$s_1=931707137$、$s_2=141729313$。源码在打印两个状态后才开始加密，因此解密流必须从 $s_2$ 继续推进一次，首个使用的是 $s_3$。

```python
from Crypto.Util.number import long_to_bytes

a = 2223895827
b = 2180283007
modulus = 3462137369
high1 = 14216
high2 = 2162
ciphertext = 405876446443434716158061994680916497969770152218293569911902716429842633796269271924

candidates = []
for low in range(1 << 16):
    s1 = (high1 << 16) | low
    s2 = (a * s1 + b) % modulus
    if s2 >> 16 == high2:
        candidates.append(s2)

if len(candidates) != 1:
    raise ValueError(f"候选状态数量异常：{len(candidates)}")

state = candidates[0]
plain = bytearray()
for value in long_to_bytes(ciphertext):
    state = (a * state + b) % modulus
    plain.append(value ^ ((state >> 16) % 10))

print(plain.decode())
```

输出：

```text
0xGame{Enumerate__att4ck_is_us3ful}
```

## 方法总结

截断 LCG 已知相邻输出时，应枚举缺失位并用下一次输出筛选，而不是枚举全部种子。恢复状态后还必须对照源码确认生成器已经推进了几次，否则会产生一位错位的密钥流。
