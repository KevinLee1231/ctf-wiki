# Sequences & Bases

## 题目简述

程序先把 flag 转成逐字节的二进制串，再分别记录所有连续 `0` 游程和连续 `1` 游程的长度。例如，二进制串中的每段 `000` 会向零游程序列加入 `3`。

两组游程长度随后分别被直接拼成数字字符串，并以“该组最大游程长度加一”为进制解释：

```python
p = int("".join(str(x) for x in zero_runs), max(zero_runs) + 1)
q = int("".join(str(x) for x in one_runs), max(one_runs) + 1)
n = p * q
```

附件只保存 $n$ 的大端字节表示。目标是先从乘积分解出可能的 $p,q$，再恢复两组游程以及原始比特串。

## 解题过程

先把密文字节解释为整数并分解 $n$。这里的 $p,q$ 并不一定是素数，所以不能只取两个素因子；需要把 $n$ 的全部素因子按重数展开，再枚举这些因子的二分分组。每种分组乘积才是一组候选 $(p,q)$。

对每个候选继续枚举小进制。真正的游程表示具有两个明显约束：

- 游程长度从 1 开始，所以进制表示中不会出现数字 `0`；
- 二进制串中的 `0` 段和 `1` 段交替出现，因此两组游程数量之差最多为 1。

最后分别尝试由 `0` 或 `1` 开始交替展开游程，按 8 位还原字符，并用已知的 `shellmates{...}` 格式验证结果。下面给出可直接在 SageMath Python 环境中运行的核心求解脚本：

```python
from functools import reduce
from operator import mul

from sage.all import factor


def to_base(value, base):
    result = ""
    while value:
        result = str(value % base) + result
        value //= base
    return result or "0"


def rebuild(zero_digits, one_digits, first):
    zero_runs = list(map(int, zero_digits))
    one_runs = list(map(int, one_digits))
    i0 = i1 = 0
    bit = first
    chunks = []

    while i0 < len(zero_runs) or i1 < len(one_runs):
        if bit == "0":
            if i0 >= len(zero_runs):
                return None
            chunks.append("0" * zero_runs[i0])
            i0 += 1
            bit = "1"
        else:
            if i1 >= len(one_runs):
                return None
            chunks.append("1" * one_runs[i1])
            i1 += 1
            bit = "0"

    bits = "".join(chunks)
    if len(bits) % 8:
        return None
    return bytes(int(bits[i:i + 8], 2) for i in range(0, len(bits), 8))


n = int.from_bytes(open("ciphertext", "rb").read(), "big")
prime_factors = []
for prime, exponent in factor(n):
    prime_factors.extend([int(prime)] * int(exponent))

pairs = set()
for mask in range(1, (1 << len(prime_factors)) - 1):
    left = reduce(
        mul,
        (f for i, f in enumerate(prime_factors) if mask >> i & 1),
        1,
    )
    pairs.add(tuple(sorted((left, n // left))))

for left, right in pairs:
    for p, q in ((left, right), (right, left)):
        for base0 in range(2, 9):
            zero_digits = to_base(p, base0)
            if "0" in zero_digits:
                continue
            for base1 in range(2, 9):
                one_digits = to_base(q, base1)
                if "0" in one_digits or abs(len(zero_digits) - len(one_digits)) > 1:
                    continue
                for first in ("0", "1"):
                    candidate = rebuild(zero_digits, one_digits, first)
                    if candidate and candidate.startswith(b"shellmates{"):
                        print(candidate.decode())
                        raise SystemExit
```

脚本恢复出：

```text
shellmates{Just_a_sm4ll_fl4g}
```

## 方法总结

本题把二进制游程编码、非标准进制表示和整数乘积分解叠在一起。关键不是把 $n$ 当作普通 RSA 模数，而是注意生成器中的 $n=pq$ 只是把两组结构化整数混合起来；分解后还要枚举素因子的分组，而不是假设 $p,q$ 各自为素数。

面对类似题目，应从生成代码提取表示层不变量，例如数字范围、禁止出现的数字、序列长度差和交替关系。它们通常能把“因子分组 × 进制 × 起始状态”的搜索空间压缩到可枚举范围，并提供比单纯检查可打印字符更强的候选验证条件。
