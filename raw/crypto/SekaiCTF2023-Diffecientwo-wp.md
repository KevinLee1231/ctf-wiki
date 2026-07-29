# Diffecientwo

## 题目简述

服务使用一个 Bloom filter 缓存用户发布的短消息：

```python
LONELY_FANS = SocialCache(2**32 - 5, 64, post_size=32, num_posts=22)
```

每条消息最多 32 字节，可以发布 22 条。加入消息时，程序用 MurmurHash3 在种子 `0..63` 下计算 64 个位置并写入集合。领取 flag 时并不检查目标宣传语是否真的发布过，只检查：

```python
check(b"#SEKAICTF #DEUTERIUM #DIFFECIENTWO #CRYPTO")
```

即目标字符串的 64 个哈希位置是否全部已经出现在集合中。目标字符串超过 32 字节，无法直接发布；但 Bloom filter 本身允许假阳性，因此可以构造若干合法短消息，使它们的哈希位置并集覆盖目标的全部位置。

## 解题过程

### 将目标改写为哈希覆盖问题

先计算目标在 64 个 seed 下的 MurmurHash3：

```python
def hashes(message: bytes):
    return [mmh3.hash(message, seed) % (2**32) for seed in range(64)]

targets = hashes(TARGET)
```

现在需要寻找不超过 22 条、每条不超过 32 字节的消息，使：

```text
targets ⊆ union(hashes(message_i))
```

暴力枚举 32 字节消息不可行，但 MurmurHash3 不是密码哈希。对固定长度消息，其内部状态由可逆的 32 位加法、乘奇数、异或移位和循环移位组成，可以用位向量约束反求消息块。

### 逆向 MurmurHash3

官方脚本先逆向输出端的 avalanche：

```text
h ^= h >> 16
h *= 0x85ebca6b
h ^= h >> 13
h *= 0xc2b2ae35
h ^= h >> 16
```

乘数在模 $2^{32}$ 下为奇数，因此可逆；异或右移也可通过位约束唯一恢复。脚本直接用 Z3 求出最终混合前的状态。

对一条由 `nb` 个 32 位块组成的消息，每个 seed 的主体更新满足：

$$
h_{i+1}=\operatorname{ROL}_{13}(h_i\oplus k_i)\cdot5+\mathtt{0xe6546b64}
\pmod {2^{32}}.
$$

选定若干 `(seed, target_hash)` 对后，为同一组消息块 `k_i` 建立多条约束：

```python
blocks = [BitVec(f"k_{i}", 32) for i in range(nb)]

for seed, target in zip(seeds, targets):
    h = BitVecVal(seed, 32)
    for block in blocks:
        h = RotateLeft(h ^ block, 13) * 5 + 0xE6546B64
    solver.add(h == reverse_finalizer(target, 4 * nb))
```

一组约束有解后，将求出的块再通过 Murmur 的块预混合逆变换还原为实际消息字节。官方脚本使用 7 个块，即每条消息 28 字节，满足长度限制。

### 分组覆盖 64 个位置

一次让同一消息命中太多指定 `(seed, hash)` 对会使约束无解，因此把 64 个目标位置分组。每组选择不同 seed，求出一条短消息，再重新计算其全部 64 个哈希；重复直到目标位置全部被覆盖。

最终得到 22 条十六进制消息。逐条调用“发布”接口后，Bloom filter 中已经包含目标宣传语的所有哈希位置：

```python
for message_hex in found_keys:
    io.sendline(b"2")
    io.sendline(message_hex)

io.sendline(b"3")
```

虽然目标字符串从未真正加入数据库，`grant_free_api()` 的成员检查仍会返回真并输出 flag。

## 方法总结

- 核心技巧：把 Bloom filter 的假阳性转化为多 seed 哈希位置覆盖，再用 Z3 逆向非密码哈希构造短消息。
- 识别信号：权限判断只调用概率型成员结构的 `check`，攻击者又能写入多个自选元素；哈希函数是 MurmurHash3 这类可建模、非抗原像算法。
- 复用要点：构造时要同时满足消息长度、写入次数、seed 和实现中的取模/有符号语义。不能只在理想化公式中验证，最终必须用服务同一 `mmh3` 实现重算所有位置并确认目标集合被完整覆盖。
