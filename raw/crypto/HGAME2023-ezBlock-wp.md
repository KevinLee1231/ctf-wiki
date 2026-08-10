# ezBlock

## 题目简述

题目实现了一个 16 位的代换网络：状态被拆成四个 4 位半字节，每一轮先异或 16 位轮密钥，再分别经过同一个 S 盒。题目给出 16 组结构化明文

```text
0000, 1111, 2222, ..., ffff
```

以及对应密文。目标是利用这些明密文对恢复五轮轮密钥，并按顺序拼成 flag。

S 盒是一个双射，且每个半字节相互独立，因此不必在整个 $2^{16}$ 密钥空间中穷举；可以分别对四条 4 位支路做差分分析。

## 解题过程

设最后一轮某个半字节的两个密文为 $c_0,c_1$，候选末轮密钥为 $k$。逆 S 盒之后的差分为

$$
S^{-1}(c_0\oplus k)\oplus S^{-1}(c_1\oplus k).
$$

另一方面，明文集合覆盖了该半字节的全部取值，所以任意一对明文的输入差分都已知。先统计 S 盒的一轮差分分布，再将分布逐轮复合，就能得到到达最后一轮之前各种差分的出现次数。对所有明密文对累加分数，得分最高的候选就是末轮密钥半字节。

确定末轮密钥后，对密文异或该密钥并套用逆 S 盒，相当于剥掉一轮。重复这一过程即可依次得到第 4、3、2、1 轮密钥；最后剩下的状态与明文异或，得到第 0 轮密钥。四条半字节支路独立处理后，再按位拼回每个 16 位轮密钥。

完整恢复脚本如下：

```python
S_BOX = {
    0: 0x6, 1: 0x4, 2: 0xC, 3: 0x5,
    4: 0x0, 5: 0x7, 6: 0x2, 7: 0xE,
    8: 0x1, 9: 0xF, 10: 0x3, 11: 0xD,
    12: 0x8, 13: 0xA, 14: 0x9, 15: 0xB,
}
INV_S_BOX = {value: key for key, value in S_BOX.items()}


def make_difference_table():
    table = {}
    for left in range(16):
        for right in range(left):
            input_diff = left ^ right
            output_diff = S_BOX[left] ^ S_BOX[right]
            row = table.setdefault(input_diff, {})
            row[output_diff] = row.get(output_diff, 0) + 1
    return table


def compose_tables(previous, one_round):
    result = {}
    for input_diff, middle_distribution in previous.items():
        row = result.setdefault(input_diff, {})
        for middle_diff, left_count in middle_distribution.items():
            for output_diff, right_count in one_round[middle_diff].items():
                row[output_diff] = row.get(output_diff, 0) + left_count * right_count
    return result


def difference_table(rounds):
    identity = {value: {value: 1} for value in range(16)}
    one_round = make_difference_table()
    table = identity
    for _ in range(rounds):
        table = compose_tables(table, one_round)
    return table


def rank_last_round_keys(cipher_nibbles, table):
    scores = {}
    for left in range(len(cipher_nibbles)):
        for right in range(left):
            input_diff = left ^ right
            for key in range(16):
                output_diff = (
                    INV_S_BOX[cipher_nibbles[left] ^ key]
                    ^ INV_S_BOX[cipher_nibbles[right] ^ key]
                )
                scores[key] = scores.get(key, 0) + table[input_diff].get(output_diff, 0)
    return sorted(scores, key=scores.get, reverse=True)


def recover_branch(plain_nibbles, cipher_nibbles):
    round_keys = {}
    state = cipher_nibbles[:]

    for removed_rounds in range(4):
        remaining_s_boxes = 3 - removed_rounds
        candidates = rank_last_round_keys(
            state,
            difference_table(remaining_s_boxes),
        )
        key = candidates[0]
        round_keys[4 - removed_rounds] = key
        state = [INV_S_BOX[value ^ key] for value in state]

    first_keys = {plain ^ value for plain, value in zip(plain_nibbles, state)}
    assert len(first_keys) == 1
    round_keys[0] = first_keys.pop()
    return round_keys


def recover_keys(plaintexts, ciphertexts):
    keys = [0] * 5
    for branch in range(4):
        shift = 4 * branch
        plain_nibbles = [(value >> shift) & 0xF for value in plaintexts]
        cipher_nibbles = [(value >> shift) & 0xF for value in ciphertexts]
        branch_keys = recover_branch(plain_nibbles, cipher_nibbles)
        for round_number, nibble in branch_keys.items():
            keys[round_number] |= nibble << shift
    return keys


plaintexts = [value * 0x1111 for value in range(16)]
ciphertexts = [
    28590, 33943, 30267, 5412,
    11529, 3089, 46924, 59533,
    12915, 37743, 64090, 53680,
    18933, 49378, 23512, 44742,
]

keys = recover_keys(plaintexts, ciphertexts)
print("hgame{" + "_".join(f"{key:x}" for key in keys) + "}")
```

运行结果为：

```text
hgame{4f42_f493_4f92_4570_d8d5}
```

## 方法总结

本题的核心是将 16 位分组拆成四个互不影响的 4 位分支，并利用 S 盒差分分布给候选轮密钥打分。每恢复一轮就逆向剥去一层，最终把指数级的整体穷举转化为若干次 16 个候选之间的比较。遇到类似小型 SPN 时，应先检查是否存在置换层或跨分支混合；如果没有，按半字节独立分析通常是最直接的突破口。
