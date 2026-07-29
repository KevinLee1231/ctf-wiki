# cryptoGRAPHy 3

## 题目简述

第三题要求在不知道密钥的情况下直接恢复原始最短路径。服务公开一个 60 节点图的全部边，并允许取得该图上所有查询的 token 链和加密响应；随后随机给出 10 组查询结果，要求在 30 秒内提交真实节点路径。题目保证每组答案唯一。

攻击模型来自论文 [An Efficient Query Recovery Attack Against a Graph Encryption Scheme](https://eprint.iacr.org/2022/838) 第 4 节：攻击者同时知道明文图和完整查询泄漏，可以分别为明文最短路径结构与密文 token 结构计算与节点编号无关的规范名称，再把两侧匹配起来。

决定性信息不是加密响应的内容，而是 token 在不同查询中的相等关系和先后连接关系。

## 解题过程

### 为明文图建立路径结构索引

对原图中的每个终点 `r`，计算所有节点到 `r` 的单终点最短路径树。为了消除节点编号的影响，对每棵有根树递归生成规范名称：

```python
def canonical_subtree(tree, node, names):
    if tree.out_degree[node] == 0:
        names[node] = "10"
        return

    children = []
    for child in tree.neighbors(node):
        canonical_subtree(tree, child, names)
        children.append(names[child])

    children.sort()
    names[node] = "1" + "".join(children) + "0"
```

对子节点名称排序后再拼接，同构的有根树会得到相同字符串。根节点名称先做 SHA-256；向路径下方移动时，再把当前子树名称与父路径名称组合：

```python
path_name[root] = H(subtree_name[root])
path_name[node] = H(subtree_name[node] + ";" + path_name[parent])
```

将每个路径名称映射到可能的真实节点对 `(source, destination)`：

```text
M[path_name] -> {(source, destination), ...}
```

### 从全部查询响应建立 token 森林

服务的“Query Responses”菜单会打乱并输出所有查询。每条记录的前半部分由初始 token 和沿路径搜索得到的后续 token 组成。将其按固定 token 长度切块后，可以恢复 token 间的父子关系：

```python
def token_edges(token_string):
    blocks = [
        token_string[i:i + TOKEN_HEX_LEN]
        for i in range(0, len(token_string), TOKEN_HEX_LEN)
    ]

    edges = []
    current = blocks[0]
    for token in blocks[1:]:
        edges.append((token, current))
        current = token
    return edges
```

把所有关系合并后得到一片以 token 为节点的有向森林。对每个入度为零的根应用与明文树完全相同的规范命名算法，得到：

```text
D[token] -> path_name
```

于是 `M[D[token]]` 就给出了该 token 对应的明文节点对。题目保证挑战路径的匹配唯一，因此不需要额外消歧。

### 回答挑战

挑战首先给出查询 token，再给出沿路径的响应 token。对每个 token 执行同一映射：

```python
answer = []

query_pair = next(iter(M[D[query_token]]))
answer.append(query_pair[0])

for token in response_tokens:
    pair = next(iter(M[D[token]]))
    answer.append(pair[0])
```

初始 token 对应源节点；后续 token 依次对应每一步的下一节点。把列表用空格连接后提交即可。

计算原图全部单终点最短路径树和规范名称较耗时，应在进入挑战菜单前完成；服务的 30 秒限制从程序启动附近开始计时，官方脚本会先收集图和查询数据，随后一次性建立两个索引并快速回答 10 题。

## 方法总结

- 核心技巧：对明文最短路径树和密文 token 森林使用同一个有根树规范化算法，以路径结构指纹完成查询恢复。
- 识别信号：攻击者知道原始图，能够观察全部查询的 token 相等关系和顺序关系，而加密方案的泄漏结构与明文最短路径树同构。
- 复用要点：树的规范名称必须对子树名称排序，避免依赖遍历顺序；名称还要包含从当前节点到根的上下文，否则相同局部子树可能错误碰撞。存在多个候选时必须进一步消歧，本题只因生成器保证唯一才可直接取集合中的元素。
