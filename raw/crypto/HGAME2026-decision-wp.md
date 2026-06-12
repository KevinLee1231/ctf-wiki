# Decision

## 题目简述

题目把 flag 去掉 `hgame{}` 后按 little-endian 转成二进制串，每一位输出一组数据。若 bit 为 1，则输出同一个 LWE 实例的 15 个样本；若 bit 为 0，则输出 15 个完全随机的 `(a, b)` 元组。参数为 `n = 25, m = 15`，单组样本数小于维度，无法仅靠一组恢复私钥，但任意两个 bit=1 的 chunk 合并后有 30 个样本，可以通过格规约恢复共同私钥 `s`。

题目定义代码如下：

```python
flagbin = bin(int.from_bytes(flag.split(b"hgame{")[1][:-1], "little"))[2:].rjust(25 * 8, "0")

n, m = 25, 15
q = getPrime(128)
F = GF(q)
D = DiscreteGaussianDistributionIntegerSampler(2**16)
lwe = LWE(n=n, q=q, D=D)

def encrypt_bit(bit):
    if bit == 1:
        samples_list = [lwe() for _ in range(m)]
        return [tuple(list(a) + [b]) for a, b in samples_list]
    else:
        return [tuple([F.random_element() for _ in range(n + 1)]) for _ in range(m)]
```

附件中的 `output.txt` 是所有 bit 对应的样本列表，体积较大，WP 只需要说明它是 200 个 chunk 的 LWE/随机样本混合数据，不需要全文贴出。

## 解题过程

先从输出中任取两个 chunk，假设它们都是 bit=1，把两组样本合并。LWE 样本满足：

```text
b = <a, s> + e (mod q)
```

其中误差 `e` 很小。构造格后 LLL 会把小误差向量约化出来，再由 `A * s = b - e` 解出私钥 `s`：

```python
def recover_s_from_chunks(chunk1, chunk2):
    samples = chunk1 + chunk2
    m = len(samples)
    A = matrix(ZZ, [x[:-1] for x in samples])
    b = vector(ZZ, [x[-1] for x in samples])

    M = block_matrix([
        [A.T, matrix.zero(n, 1)],
        [q * matrix.identity(m), matrix.zero(m, 1)],
        [matrix(ZZ, -b), matrix.identity(1)],
    ])

    for row in M.LLL():
        if abs(row[-1]) != 1:
            continue
        if abs(row[0]) >= 2**30:
            continue

        e = vector(F, row[:-1]) * (-row[-1])
        try:
            return A.solve_right(vector(F, b) - e)
        except Exception:
            pass
```

遍历前若干 chunk 对，直到找到能解出 `s` 的组合：

```python
real_s = None
for i in range(20):
    for j in range(i + 1, 20):
        real_s = recover_s_from_chunks(out[i], out[j])
        if real_s is not None:
            break
    if real_s is not None:
        break
```

有了 `s` 后，对每个 chunk 取第一条样本计算误差：

```python
error = b - (a * real_s)
err_val = int(error)
if err_val > q // 2:
    err_val -= q

bit = "1" if abs(err_val) < 2**60 else "0"
```

LWE 样本对应小误差，随机样本对应模 `q` 上的随机误差。恢复所有 bit 后按题目 little-endian 方式转回字节即可得到 flag 内容。

## 方法总结

- 核心技巧：把“bit=1 输出 LWE、bit=0 输出随机”的判别问题转成先找两组 LWE 样本恢复共同私钥，再逐组判误差大小。
- 识别信号：多组样本共用同一 LWE 私钥，且单组样本不足维但两组足够时，应尝试组合 chunk 做格恢复。
- 复用要点：WP 中保留 LWE 参数和样本生成逻辑即可，超长输出文件只描述形态和用法，不应全文写入。
