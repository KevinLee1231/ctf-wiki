# Some Trick

## 题目简述

题目把整数按 $8209$ 进制拆成 $79$ 位，并在每一位上使用一个长度为 $8209$ 的置换。所有置换由程序开头打印的 `CIPHER_SUITE` 作为 Python `random` 种子生成，因此攻击者能够完整重建。

核心函数为：

```python
def enc(k, m, G):
    if not G:
        return m
    mod = len(G[0])
    return (
        gexp(G[0], k % mod)[m % mod]
        + enc(k // mod, m // mod, G[1:]) * mod
    )
```

服务依次输出：

```python
bob_encr   = enc(FLAG, bob_key, G)
alice_encr = enc(bob_encr, alice_key, G)
bob_decr   = enc(alice_encr, bob_key, inverse(G))
```

FLAG 两侧还拼接了随机比特，恢复的整数中不能直接按字节从首位读取。

## 解题过程

### 重建置换

`gen` 将 `1..8208` 随机排列后首尾连接到 `0`，生成一个覆盖全部元素的单环置换。读取版本号中的种子后，按相同顺序调用 `random.sample`，即可重建全部 $79$ 个置换及其逆置换：

```python
CIPHER_SUITE = int(version.split("4.0.")[1])
random.seed(CIPHER_SUITE)
G = [gen(8209) for _ in range(79)]
G_inv = [inverse(g) for g in G]
```

### 先恢复两个随机 key

若指数 `k` 和输出 `ct` 已知，每一位都可以直接在 `gexp(g, k_digit)` 中反查输入位置：

```python
def dec_known_exponent(k, ct, G):
    if not G:
        return 0
    n = len(G[0])
    digit = list(gexp(G[0], k % n)).index(ct % n)
    return digit + n * dec_known_exponent(k // n, ct // n, G[1:])
```

于是：

```python
alice_key = dec_known_exponent(bob_encr, alice_encr, G)
bob_key = dec_known_exponent(alice_encr, bob_decr, G_inv)
```

### 由已知输入恢复 FLAG

现在 `bob_encr = enc(FLAG, bob_key, G)` 中只有指数 `FLAG` 未知。由于每个 `g` 都是长度 $8209$ 的单环，逐位枚举 $0\ldots8208$，找到把 `bob_key` 当前位映射为 `bob_encr` 当前位的迭代次数即可：

```python
def solve_exponent_digit(g, plain_digit, cipher_digit):
    base = tuple(g)
    for k in range(8209):
        if g[plain_digit] == cipher_digit:
            return k
        g = tuple(base[i] for i in g)
    raise ValueError("digit not found")
```

递归组合所有进制位后得到带随机前后缀的整数。最后在二进制串中定位 `SEKAI{` 的比特模式，并尝试不同右移位数；当解出的字节串以 `}` 结尾时即得到完整 flag：

```python
marker = bin(bytes_to_long(b"SEKAI{"))[2:]
offset = bin(recovered)[2:].index(marker)

for shift in range(recovered.bit_length()):
    candidate = long_to_bytes(recovered >> shift)
    if candidate.startswith(b"SEKAI{") and candidate.endswith(b"}"):
        print(candidate)
```

仓库中服务端使用的 flag 为：

```text
SEKAI{7c124c1b2aebfd9e439ca1c742d26b9577924b5a1823378028c3ed59d7ad92d1}
```

## 方法总结

- 核心技巧：利用公开 PRNG 种子重建置换，再从两组已知指数关系恢复随机 key，最终逐进制位求置换离散对数。
- 识别信号：协议把“密钥”和“消息”交替作为下一轮的指数或输入，但所有群作用都可重建，且转录中出现足够多的已知输入输出关系。
- 复用要点：分析自定义可交换加密时，应先把递归整数运算拆成独立进制位；随机前后缀不等于安全，只要明文有稳定标记，就能在比特层重新对齐。
