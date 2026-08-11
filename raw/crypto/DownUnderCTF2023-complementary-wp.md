# DownUnderCTF 2023 complementary Writeup

## 题目简述

生成脚本把 flag 从中间分为两半，将两段分别按大端整数解释为 $m_1$、$m_2$，只公开乘积 $n=m_1m_2$。题目名称暗示应从 $n$ 的互补因子对恢复原文。

## 解题过程

附件中的公开值为：

```text
6954494065942554678316751997792528753841173212407363342283423753536991947310058248515278
```

枚举 $n$ 的正因子 $d$，则另一半必为 $n/d$。把两者分别转回字节并拼接，用已知前缀 `DUCTF` 筛选正确顺序即可。下面的 Sage 脚本可以直接完成恢复：

```python
from Crypto.Util.number import long_to_bytes

n = 6954494065942554678316751997792528753841173212407363342283423753536991947310058248515278

for divisor in divisors(n):
    left = long_to_bytes(int(divisor))
    right = long_to_bytes(int(n // divisor))
    candidate = left + right
    if candidate.startswith(b"DUCTF{"):
        print(candidate.decode())
        break
```

输出为：

```text
DUCTF{is_1nt3ger_f4ct0r1s4t10n_h4rd?}
```

## 方法总结

这里没有使用难以分解的标准 RSA 模数结构，直接枚举 Sage 给出的因子即可。恢复整数编码字符串时要注意字节序、分段顺序和可能丢失的前导零；本题可借助固定 flag 前缀消除歧义。
