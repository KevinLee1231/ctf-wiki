# save-trees

## 题目简述

服务把 `0x10000000 || SHA256(flag)` 解释成一个大整数作为树根。每个内部节点按 16 位切分，并用同一个 key 异或生成左右孩子：

$$
L=(P\gg16)\oplus K,\qquad R=(P\bmod2^{16})\oplus K.
$$

服务公开树边和所有叶子值。key 的高位固定为字符串 `save thr trees!!`，只有最低 16 位随机，且范围仅为 $[2^{15},2^{16})$。

## 解题过程

根据边表建立邻接表，区分每个节点按编号排序后的左、右孩子。对每个 15 位后缀候选，从叶子向根递归还原：

```python
from Crypto.Util.number import bytes_to_long

known = bytes_to_long(b"save thr trees!!") << 16

def rebuild(node: int, parent: int) -> None:
    children = sorted(child for child in graph[node] if child != parent)
    for child in children:
        rebuild(child, node)
    if children:
        left, right = children
        values[node] = ((values[left] ^ key) << 16) + (values[right] ^ key)

for suffix in range(1 << 15, 1 << 16):
    key = known + suffix
    values = [0] * 2048
    for node, value in published_leaves:
        values[node] = value
    rebuild(0, -1)
    if hex(values[0]).startswith("0x10000000"):
        answer = values[0]
        break
```

根节点的固定十六进制前缀为候选提供强校验。把恢复出的根整数在 5 秒内提交，服务返回：

```text
tjctf{tR33s_g1v3_0xYg3ndx0x<3}
```

## 方法总结

- 这棵树不是单向哈希结构；左右孩子保留了父节点的高低部分，用已知 key 可完全逆转。
- key 只有 32768 个候选，固定根前缀足以快速筛选；不需要猜 flag 或逆 SHA-256。
- 网络脚本应先解析全部边和叶子，再在本地枚举，最后只提交一个通过结构校验的根值。
