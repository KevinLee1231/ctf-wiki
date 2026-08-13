# GreyCTF2024 Maze Runner WP

## 题目简述

服务连续给出 50 个逐渐变大的迷宫，并允许有限次“穿墙”。每轮要回答从左上角到右下角的最短步数。穿过普通通道和穿过墙都消耗一步，但后者还会消耗一次 wall-phase，因此搜索状态必须包含剩余能力次数。

## 解题过程

把状态定义为三元组 $(r,c,p)$，其中 $(r,c)$ 是房间坐标，$p$ 是剩余穿墙次数。对四个方向扩展：没有墙时进入 $(r',c',p)$，有墙时只有 $p>0$ 才能进入 $(r',c',p-1)$。因为每条移动边的步数代价都是 1，直接使用 BFS：

```python
from collections import deque

def shortest(walls, phases):
    n = len(walls)
    q = deque([(0, 0, phases, 0)])
    seen = {(0, 0, phases)}
    delta = [(-1, 0), (1, 0), (0, 1), (0, -1)]

    while q:
        r, c, left, dist = q.popleft()
        if (r, c) == (n - 1, n - 1):
            return dist
        for d, (dr, dc) in enumerate(delta):
            nr, nc = r + dr, c + dc
            if not (0 <= nr < n and 0 <= nc < n):
                continue
            need = int(walls[r][c][d])
            state = (nr, nc, left - need)
            if state[2] >= 0 and state not in seen:
                seen.add(state)
                q.append((*state, dist + 1))
```

解析文本迷宫时，房间之间的竖线决定左右墙，横线决定上下墙；相邻房间的墙信息要同步写入两个方向。每轮把 BFS 返回的距离发回服务，完成 50 轮后得到：

```text
grey{g1ad3rs_pha5erS_y0u_hAvE_jo1n3d_tH3_m4ze_eSc4p3rs!}
```

## 方法总结

“有限次破墙”的迷宫不能只用二维 `visited[r][c]`，因为到达同一格但剩余能力不同，会影响后续可达性。将资源余量加入状态后，问题重新变成无权图最短路，BFS 即可保证首次到达终点时的步数最小。
