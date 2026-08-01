# Sus Box

## 题目简述

题目公开一张随机置换 S-box，并用同一把 16 字节主密钥逐块加密。每个字节位置独立执行：

$$
c_i=S(p_i\oplus k_{0,i})\oplus k_{1,i},\qquad k_1=\operatorname{MD5}(k_0).
$$

明文以已知字符串 `I have a secret, please don't share: ` 开头，正好提供前两个 16 字节块的已知明密文对。算法只有一层 S-box、没有跨字节扩散，因此可以对每个位置做差分分析，再用 $k_1=MD5(k_0)$ 的关系筛选整把密钥。

## 解题过程

取两个已知块 $P_1,P_2$ 及其密文 $C_1,C_2$。对相同字节位置，末尾异或的 $k_1$ 会在差分中消失：

$$
C_{1,i}\oplus C_{2,i}
=S(P_{1,i}\oplus k_{0,i})\oplus S(P_{2,i}\oplus k_{0,i}).
$$

先枚举 S-box 的全部输入对 $(u,v)$，按照

$$
(u\oplus v,\ S(u)\oplus S(v))
$$

建立差分分布表。对第 $i$ 个字节，已知输入差分 $P_{1,i}\oplus P_{2,i}$ 和输出差分 $C_{1,i}\oplus C_{2,i}$，便能查出可能的 S-box 输入 $u=P_{1,i}\oplus k_{0,i}$，从而得到该位置的候选 $k_{0,i}=P_{1,i}\oplus u$。

```python
# ddt[delta_in][delta_out] 保存可能的 S-box 输入字节
ddt = [[set() for _ in range(256)] for _ in range(256)]
for u in range(256):
    for v in range(256):
        ddt[u ^ v][sbox[u] ^ sbox[v]].update((u, v))

candidates = []
for p1, p2, c1, c2 in zip(P1, P2, C1, C2):
    candidates.append(ddt[p1 ^ p2][c1 ^ c2])
```

对 16 个位置的候选集合做笛卡尔积。每个组合先还原 $k_0$，再计算 $k_1=MD5(k_0)$，逆向第一块：

```python
for sbox_inputs in itertools.product(*candidates):
    k0 = bytes(p ^ u for p, u in zip(P1, sbox_inputs))
    k1 = hashlib.md5(k0).digest()
    trial = bytes(
        sbox_inv[c ^ b] ^ a
        for c, a, b in zip(C1, k0, k1)
    )
    if trial == P1:
        key = k0
        break
```

官方实例约有 25165824 个整钥候选，验证第一块后只剩正确密钥。按 `xor k1 → inverse S-box → xor k0` 解密全部密文，得到：

```text
byuctf{if_you_used_a_llm_youre_missing_out_learning_a_really_cool_attack_!}
```

## 方法总结

- 核心技巧：对无扩散的一轮 SPN 建立 S-box 差分表，逐字节缩小轮密钥候选，再利用派生密钥关系验证。
- 识别信号：相同密钥独立处理每个字节、公开 S-box、两块已知明文以及没有置换/混合层，说明差分不会在字节间扩散。
- 复用要点：差分只消除了末尾轮密钥，不能直接给出唯一 $k_0$；需要用完整块验证或密钥调度关系消歧。DDT 应保存集合，不能假设一个差分只有一组输入。
