# Wiki Game

## 题目简述

题目把 Wikipedia 页面抽象为一个有向图：每篇文章是一个顶点，页面中的链接是一条有向边。每组数据给出顶点数 $n$、边数 $m$、起点 `src` 和终点 `dst`，要求判断能否从 `src` 出发，在至多 $6$ 条边内到达 `dst`。

约束为 $2\le n\le 1000$、$1\le m\le 4000$，测试组数不超过 $20$。边有方向，反向可达不代表正向可达。

## 解题过程

这是一个带深度上限的可达性问题。对每组数据建立邻接表，从 `src` 做广度优先搜索，并在队列中同时保存当前点和从起点走过的边数：

1. 将 `(src, 0)` 入队并标记 `src`。
2. 每次取出 `(u, d)`；若 `u == dst`，直接输出 `YES`。
3. 当 $d=6$ 时不再扩展该节点，否则把尚未访问的出邻点以距离 $d+1$ 入队。
4. 队列清空仍未遇到 `dst`，输出 `NO`。

广度优先搜索首次访问某个顶点时已经取得最短距离，因此同一顶点无需重复入队。参考实现如下：

```python
T = int(input())

for _ in range(T):
    n, m = map(int, input().split())
    graph = [[] for _ in range(n)]
    for _ in range(m):
        u, v = map(int, input().split())
        graph[u].append(v)

    src, dst = map(int, input().split())
    queue = [(src, 0)]
    head = 0
    visited = {src}
    reachable = False

    while head < len(queue):
        u, distance = queue[head]
        head += 1

        if u == dst:
            reachable = True
            break
        if distance == 6:
            continue

        for v in graph[u]:
            if v not in visited:
                visited.add(v)
                queue.append((v, distance + 1))

    print("YES" if reachable else "NO")
```

每组数据的时间复杂度为 $O(n+m)$，空间复杂度为 $O(n+m)$。

## 方法总结

题目的 Wikipedia 背景不改变模型，本质是“有向图上距离不超过常数的可达性”。关键是保留边的方向，并在 BFS 中于距离达到 $6$ 时停止扩展；不能把图当成无向图，也不能先无界搜索再凭路径存在性作答。
