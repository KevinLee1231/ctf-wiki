# theartofwar

## 题目简述

附件把同一个无填充明文整数 $m$ 用相同的小素数指数 $e$，分别在 $e$ 个独立 RSA 模数下加密，公开所有 $(n_i,c_i)$，其中 $c_i=m^e\bmod n_i$。大量互素模数和重复明文满足 Håstad 广播攻击的条件。

## 解题过程

对所有同余式

$$C\equiv c_i\pmod{n_i}$$

使用中国剩余定理，可得到模 $N=\prod_i n_i$ 意义下唯一的 $C$。这里样本数量就是 $e$，且每个模数约 512 位；flag 明文远短于模数乘积所容纳的范围，因此 $m^e<N$，CRT 结果不再只是同余类，而恰好等于整数 $m^e$。对它取精确的 $e$ 次整数根即可恢复明文。

```python
from sympy.ntheory.modular import crt
from Crypto.Util.number import long_to_bytes

moduli = []
ciphertexts = []
with open("output.txt", "r", encoding="utf-8") as f:
    for line in f:
        name, _, value = line.split()
        if name == "e":
            exponent = int(value)
        elif name.startswith("n"):
            moduli.append(int(value))
        elif name.startswith("c"):
            ciphertexts.append(int(value))

power = int(crt(moduli, ciphertexts)[0])

def integer_nth_root(value: int, degree: int) -> int:
    low, high = 0, 1
    while high ** degree <= value:
        high *= 2
    while low + 1 < high:
        middle = (low + high) // 2
        if middle ** degree <= value:
            low = middle
        else:
            high = middle
    return low

message = integer_nth_root(power, exponent)
assert message ** exponent == power
print(long_to_bytes(message).decode())
```

脚本恢复出：

```text
tjctf{the_greatest_victory_is_that_which_require_no_battle}
```

## 方法总结

- 核心技巧：对相同明文、相同小指数和多个互素模数应用 CRT，再取精确整数根。
- 识别信号：无 padding 的 RSA、重复明文、固定小 $e$，并给出至少 $e$ 组独立密文。
- 复用要点：取根后必须验证 $m^e=C$；若不精确，应检查模数是否互素、样本是否足够或明文是否经过随机 padding。
