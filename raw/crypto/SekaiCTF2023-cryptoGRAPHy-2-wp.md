# cryptoGRAPHy 2

## 题目简述

本题沿用 cryptoGRAPHy 1 的图加密方案，但不再公开密钥。服务每轮生成一个含 130 个节点的连通随机图，指定一个终点 `dest`，允许最多 130 次查询，然后要求提交“所有节点到 `dest` 的单终点最短路径树”中各节点度数的升序序列。

攻击者扮演 honest-but-curious 服务器：按协议执行查询，却会比较不同查询中泄露的确定性 token。GES 虽然把路径节点对加密，但底层字典搜索是 response-revealing 的；`GES.search` 会把沿途 token 链直接返回。因此，对同一终点发起足够多查询时，可以从 token 的相等关系恢复最短路径树的结构。

该泄漏对应 [Efficient Graph Encryption Scheme for Shortest Path Queries](https://dl.acm.org/doi/10.1145/3433210.3453099) 中的 Query Pattern、Path Intersection Pattern 和路径长度泄漏。本文下面直接给出它们在本题中的利用方式。

## 解题过程

### 查询同一终点的全部来源

对固定终点 `dest`，从每个其它节点 `u` 查询 `(u, dest)`。服务返回：

```text
初始查询 token
沿最短路径产生的后续 token
加密路径响应
```

恢复树只需要 token，不需要解密后半部分响应。将每条 token 串按 32 字节切分，并去掉末尾代表 `(dest, dest)` 的公共 token。

假设最短路径是：

```text
u -> w -> dest
```

查询 `(u, dest)` 得到的 token 链中，第二个路径 token 会与查询 `(w, dest)` 的第一个路径 token相同。这是因为二者都表示相同的节点对标签 `(w, dest)`。因此，token 相等就能把两条路径的公共后缀合并。

### 从末端向根重建树

服务随机化了真实节点标签，但题目最终只要求度数序列，所以可以给恢复出的节点分配任意临时编号。

先处理每条链最后一个非终点 token。它们都对应与 `dest` 直接相邻的树节点：

```python
token_node = {}
tree = nx.Graph()
next_id = dest + 1

for chain in chains:
    if len(chain) < 32:
        continue
    token = chain[-32:]
    if token not in token_node:
        token_node[token] = next_id
        tree.add_edge(next_id, dest)
        next_id += 1
```

随后按“倒数第二块、倒数第三块……”向前扫描。新 token 对应的节点，其父节点就是同一链中右侧相邻 token 已映射的节点：

```python
max_depth = max(len(chain) // 32 for chain in chains)

for depth in range(2, max_depth + 1):
    for chain in chains:
        if len(chain) < depth * 32:
            continue

        current = chain[-depth * 32:-(depth - 1) * 32]
        parent = chain[-(depth - 1) * 32:-(depth - 2) * 32]

        if current not in token_node:
            token_node[current] = next_id
            tree.add_edge(next_id, token_node[parent])
            next_id += 1
```

这里恢复的是无标签树，但节点度数不依赖真实编号。最终提交：

```python
answer = " ".join(
    map(str, sorted(tree.degree(node) for node in tree.nodes))
)
```

每轮需要把 129 个来源都查询一遍，再用 `-1` 结束查询并提交结果；连续完成 10 轮即可通过。

## 方法总结

- 核心技巧：利用确定性 token 暴露的路径交集模式，把同一终点的全部最短路径合并成一棵无标签树。
- 识别信号：加密查询隐藏了响应内容，却返回可跨查询比较的 token 链；服务又允许对同一终点枚举所有来源。
- 复用要点：若目标只要求度数、拓扑或同构类，就不必恢复真实节点标签。处理共享后缀时从靠近根节点的一端开始，可保证父节点在子节点之前被建立。
