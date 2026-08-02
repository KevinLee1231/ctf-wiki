# cheapest-cookies-2

## 题目简述

服务端连续生成 50 张含 21 个点、40 条无向带权边的图。每轮需在 3 秒内回答从结点 0 到结点 20 的最短距离，不可达时输出 `-1`。这是图算法题，raw 目录没有独立 programming 分类，因此归入 `_unclassified`。

## 解题过程

边权均为 1 到 20 的正整数，使用 Dijkstra 即可。每轮读入 40 行 `u v w`，建立双向邻接表，从 0 出发维护最短距离：

```python
from heapq import heappop, heappush

def shortest(edges):
    g = [[] for _ in range(21)]
    for u, v, w in edges:
        g[u].append((v, w))
        g[v].append((u, w))

    inf = 10**18
    dist = [inf] * 21
    dist[0] = 0
    pq = [(0, 0)]
    while pq:
        d, u = heappop(pq)
        if d != dist[u]:
            continue
        for v, w in g[u]:
            nd = d + w
            if nd < dist[v]:
                dist[v] = nd
                heappush(pq, (nd, v))
    return -1 if dist[20] == inf else dist[20]
```

网络脚本应以服务端的 `correct answer 50` 解析实际轮数，并在每次提示后立即回传结果。50 轮通过后得到：

```text
tjctf{w00_w3_have_th3_c00k1es_n0w}
```

## 方法总结

- 正权图最短路使用 Dijkstra，复杂度 $O((V+E)\log V)$，远低于三秒限制。
- 无向边必须同时加入两个方向；不可达状态要按协议转换为 `-1`。
- 交互算法题的正确性还取决于稳定解析轮次与提示边界，不能只验证本地算法函数。
