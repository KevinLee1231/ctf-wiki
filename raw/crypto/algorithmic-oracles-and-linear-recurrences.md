# CTF Crypto - Algorithmic Oracles and Linear Recurrences

## 阅读定位

- 本卷收纳不依赖特定密码原语、但以自适应查询、序列识别或代数递推恢复状态为主障碍的算法型材料。
- 如果反馈来自加密、填充、MAC 或密文格式，优先转向对应的密码 oracle 专题；如果主要障碍是交互协议或二进制利用，也不应从本卷起步。

## Table of Contents

- [OEIS Sequence Lookup Automation for Recurrence Puzzles (X-MAS CTF 2018)](#oeis-sequence-lookup-automation-for-recurrence-puzzles-x-mas-ctf-2018)
- [Matrix Exponentiation for Fibonacci-Like Recurrences (Pwn2Win 2018)](#matrix-exponentiation-for-fibonacci-like-recurrences-pwn2win-2018)
- [Tribonacci Recurrence for Frog Jump Counting (FireShell 2019)](#tribonacci-recurrence-for-frog-jump-counting-fireshell-2019)
- [Levenshtein Distance Oracle Attack (SunshineCTF 2016)](#levenshtein-distance-oracle-attack-sunshinectf-2016)

---

## OEIS Sequence Lookup Automation for Recurrence Puzzles (X-MAS CTF 2018)

```python
import requests
from pyquery import PyQuery as pq

def lookup_next_term(seq):
    response = requests.get(
        "https://oeis.org/search",
        params={"q": ",".join(map(str, seq))},
        timeout=10,
    )
    response.raise_for_status()
    doc = pq(response.text)
    return doc("pre").eq(1).text().split(",")[len(seq)]
```

OEIS 适合识别已经收录的整数序列，但不能把首个搜索结果直接当答案。应核对偏移量、符号、缩放和至少数个后续项；若服务还包含 CAPTCHA、PoW 或 socket framing，则先把交互包装自动化，再批量查询和验证。

**关键结论：** 将已知前缀标准化后交给序列数据库，可以快速提出候选递推；真正可靠的判定来自多项复算，而不是一次搜索命中。

**参考：** X-MAS CTF 2018 — A Weird List of Sequences, writeup 12683

---

## Matrix Exponentiation for Fibonacci-Like Recurrences (Pwn2Win 2018)

```python
MOD = 10**9 + 7

def matmult(a, b):
    return (
        (a[0] * b[0] + a[1] * b[2]) % MOD,
        (a[0] * b[1] + a[1] * b[3]) % MOD,
        (a[2] * b[0] + a[3] * b[2]) % MOD,
        (a[2] * b[1] + a[3] * b[3]) % MOD,
    )

def matpow(matrix, exponent):
    result = (1, 0, 0, 1)
    while exponent:
        if exponent & 1:
            result = matmult(result, matrix)
        matrix = matmult(matrix, matrix)
        exponent >>= 1
    return result
```

任意固定阶线性递推都可以写成状态向量乘转移矩阵。遇到极大的 `N`、模环和 Fibonacci、Lucas、Tribonacci 或线性计数器时，应先构造转移矩阵，再用二进制快速幂把复杂度从 `O(N)` 降到 `O(log N)`。

**关键结论：** 不要逐项生成巨大下标的线性递推；确认初始状态、系数顺序和模数后，用矩阵快速幂并正向复算小下标样本。

**参考：** Pwn2Win CTF 2018 — Too Slow, writeup 12501

---

## Tribonacci Recurrence for Frog Jump Counting (FireShell 2019)

```python
def tribonacci(n, modulus=13371337):
    a, b, c = 0, 0, 1
    for _ in range(n):
        a, b, c = b, c, (a + b + c) % modulus
    return c
```

“每次可以走 1 到 `k` 步，求到达第 `N` 阶的方法数”会自然形成 `k` 阶线性递推。服务给出的 `N` 较小时可以预计算并跨请求缓存；`N` 很大时则沿用上一节的伴随矩阵快速幂。

**关键结论：** 先从允许步长写出递推和初值，再根据最大查询规模选择缓存或矩阵快速幂；不要把网络交互成本和数列计算成本混在一起。

**参考：** FireShell CTF 2019 — Frogs, writeup 12961

---

## Levenshtein Distance Oracle Attack (SunshineCTF 2016)

Oracle 返回猜测字符串与 secret 之间的编辑距离。恢复流程：

1. **确定长度：** 提交空字符串，距离就是 secret 长度。
2. **统计字符：** 对每个候选字符提交同长度的重复串，`length - distance` 是该字符在 secret 中的出现次数。
3. **定位位置：** 用已知存在和已知不存在的字符填充两半位置，根据距离变化二分缩小位置集合。

```python
import string

characters = {}
for candidate in string.printable:
    distance = oracle(candidate * length)
    count = length - distance
    if count > 0:
        characters[candidate] = count
```

**关键结论：** 编辑距离本身就是高信息量侧信道。先恢复长度和字符多重集，再按位置二分，可以用约 `O(n log n)` 次查询恢复 secret；实践中还要先测重复请求的稳定性和 rate limit。

**参考：** SunshineCTF 2016 — Levenshtein distance oracle

