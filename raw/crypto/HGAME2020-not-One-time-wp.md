# not_One-time

## 题目简述

服务每次都用新随机密钥逐字节异或同一个 flag，看似满足“一次一密”；问题在于每个密钥字节只从 `ascii_letters + digits` 这 62 个字符中选择，而不是覆盖全部 256 个字节。反复取得密文后，可以对每个位置的可能明文集合求交集。

## 解题过程

对某个密文字节 $c$，密钥空间为 $K$ 时，该位置所有可能明文为：

$$
P(c)=\{c\oplus k\mid k\in K\}.
$$

每次连接都会得到新的密钥和密文 $c_1,c_2,\ldots$，但明文固定，因此真实明文字节一定在 $P(c_1)\cap P(c_2)\cap\cdots$ 中。随着样本增多，错误候选会逐渐被排除。

以下代码从已经采集并按行保存的 Base64 密文中恢复明文；采集阶段只需反复连接原服务并保存每次返回值，过期的比赛地址没有保留在题解中：

```python
import base64
import string

keyspace = (string.ascii_letters + string.digits).encode()

with open("ciphertexts.txt", "r", encoding="ascii") as f:
    samples = [base64.b64decode(line.strip()) for line in f if line.strip()]

assert samples
length = len(samples[0])
assert all(len(sample) == length for sample in samples)

candidates = [set(string.printable.encode()) for _ in range(length)]

for sample in samples:
    for i, byte in enumerate(sample):
        candidates[i] &= {byte ^ key for key in keyspace}

# 已知 flag 格式可作为一致性检查，也能加快收敛。
known_prefix = b"hgame{"
for i, byte in enumerate(known_prefix):
    candidates[i] &= {byte}
candidates[-1] &= {ord("}")}

for i, values in enumerate(candidates):
    if len(values) != 1:
        raise RuntimeError(f"position {i} has candidates: {sorted(values)}")

print(bytes(next(iter(values)) for values in candidates).decode())
```

通常需要几十到上百组样本才能让每个位置都收敛为单值。官方 PDF 没有记录最终输出；公开的[参赛者复盘](https://www.cnblogs.com/wh201906/p/12232610.html)给出的恢复结果为：

```text
hgame{r3us1nG+M3$5age-&&~rEduC3d_k3Y-5P4Ce}
```

## 方法总结

- 核心技巧：密钥虽不复用，但密钥字节分布缺少大量可能值；多次观测后对候选明文集合求交即可泄露固定消息。
- 识别信号：服务重复加密同一明文，密钥来自可打印字符、字母数字或其他明显小于 256 的集合。
- 复用要点：真正安全的 OTP 要求密钥均匀覆盖完整字节空间、与消息等长且只使用一次；“每次换密钥”本身并不足够。
