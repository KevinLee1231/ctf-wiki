# GreyCTF2024 13-piece puzzle WP

## 题目简述

附件给出 13 块带固定白色定位点的棋盘拼片，要求找出铺满 $8\times8$ 棋盘的全部方案。最终答案不是某一张拼图，而是把所有方案按指定格式序列化、排序后计算 SHA-256。

## 解题过程

先为每块拼片生成四次旋转及其水平翻转，并去掉对称导致的重复朝向。只保留定位点落在棋盘白格、整块不越界的放置方式。对每个合法放置建立二进制变量 $x_{i,o,y,x}$：

- 对每块拼片 $i$，所有合法放置变量之和等于 1；
- 对棋盘每个格子，覆盖它的放置变量之和等于 1。

这就是一个精确覆盖型 MILP：

$$
\sum_{o,y,x}x_{i,o,y,x}=1
$$

$$
\sum_{(i,o,y,x)\text{ covers }c}x_{i,o,y,x}=1
$$

求得一个方案后，加入阻断约束，要求当前选中的 13 个变量下次至少有一个改变：

```python
chosen = [v for v in placement_vars if round(mip.get_value(v)) == 1]
mip.add_constraint(sum(chosen) <= len(chosen) - 1)
```

持续求解直到模型不可行，共枚举出 340 个方案。每个方案按拼片原始顺序编码为 `(朝向编号, 定位点行, 定位点列)`，三个数直接转成字节；再把 340 个字节串按十六进制字典序排序、连接并哈希：

```python
from hashlib import sha256

encoded = []
for solution in solutions:
    triples = [(orientation, row, col) for orientation, row, col in solution]
    encoded.append(bytes(v for triple in triples for v in triple).hex())

payload = b"".join(bytes.fromhex(s) for s in sorted(encoded))
print(sha256(payload).hexdigest())
```

结果为：

```text
grey{ffd26ed19381530cd0484ec1d32e0188d8801208a942a966043bb492b23c6f92}
```

## 方法总结

本题容易遗漏的不是“找到一个拼法”，而是完整枚举和确定性序列化。精确覆盖约束保证每块只用一次、每格恰好覆盖一次；阻断约束保证无重复枚举；固定拼片顺序、方案排序和字节编码则保证最终摘要可复现。
