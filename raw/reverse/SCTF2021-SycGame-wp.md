# SycGame

## 题目简述

服务连续生成 5 局 $20\times20$ 推箱子，每局有 4 个箱子且总超时 30 秒。地图没有用字符打印，而是输出 400 个整数：

- 正整数中的质数是墙；
- 合数以及 1 是可行走地面；
- `-1` 是箱子，`-2` 是玩家，`-3` 是目标；
- 移动字符为 `w/s/a/d`，分别对应上、下、左、右。

源码中的欧拉筛把合数标记在数组 `v` 中，移动逻辑以 `v[cell]` 判断可走性。四个箱子全部压到目标后本局返回成功，完成五局才执行 `cat flag`。决定性障碍是恢复这套数值地图语义并自动求解随机 Sokoban，因此归入 Reverse。

## 解题过程

### 解析地图

收到 `gift:` 后读取恰好 400 个整数并重排。正数可用普通素数判定；特殊负数需要先单独处理，不能直接作为筛数组下标：

```python
from math import isqrt

def is_prime(value):
    if value < 2:
        return False
    for divisor in range(2, isqrt(value) + 1):
        if value % divisor == 0:
            return False
    return True

grid = [numbers[i:i + 20] for i in range(0, 400, 20)]
walls = set()
boxes = set()
box_origins = set()
goals = set()
player = None

for row in range(20):
    for col in range(20):
        value = grid[row][col]
        pos = (row, col)
        if value == -1:
            boxes.add(pos)
            box_origins.add(pos)
        elif value == -2:
            player = pos
        elif value == -3:
            goals.add(pos)
        elif is_prime(value):
            walls.add(pos)
```

程序用备份数组 `ya` 保存原始负数标记，这带来两个偏离标准 Sokoban 的实现缺陷。第一，玩家直接走向 `-3` 时会把负数用于 `v[a[next]]`，实际表现为目标格不可通行，只有箱子能被推入目标。第二，执行“推箱子”后程序用 `ya[player]` 恢复玩家原位置；若玩家当时站在初始箱位，该格会重新变成 `-1`，凭空生成一个箱子。求解器因此既要保留独立的 `goals`，也要记录 `box_origins`，并禁止从初始箱位发起推动。

### 按“推箱子动作”搜索

逐步枚举四个移动会产生大量只改变玩家位置的状态。更合适的做法是把一次“推箱子”作为搜索边：

1. 在固定箱子集合下，对玩家做一次 flood fill，得到所有不穿墙、不穿箱子、也不踩目标的可达格，并保存到每格的最短步行串。
2. 对每个箱子和四个方向，检查玩家能否到达箱子背面、背面不是初始箱位，且箱子前方不是墙或另一个箱子。
3. 新状态中箱子前移一格，玩家停在箱子原位置；边上的操作串为“走到背面 + 推动方向字符”。
4. 以 `(player, sorted(boxes))` 去重，直到 `boxes == goals`。

核心搜索如下：

```python
from collections import deque

DIRECTIONS = [(-1, 0, "w"), (1, 0, "s"), (0, -1, "a"), (0, 1, "d")]

def walking_paths(start, walls, boxes, goals):
    paths = {start: ""}
    queue = deque([start])
    blocked = walls | boxes | goals
    while queue:
        row, col = queue.popleft()
        for dr, dc, key in DIRECTIONS:
            nxt = (row + dr, col + dc)
            if nxt in blocked or nxt in paths:
                continue
            paths[nxt] = paths[(row, col)] + key
            queue.append(nxt)
    return paths

def solve(player, boxes, goals, walls, box_origins):
    start = (player, tuple(sorted(boxes)))
    queue = deque([(player, frozenset(boxes), "")])
    seen = {start}

    while queue:
        player, boxes, moves = queue.popleft()
        if boxes == goals:
            return moves

        reachable = walking_paths(player, walls, boxes, goals)
        for box_row, box_col in boxes:
            for dr, dc, key in DIRECTIONS:
                behind = (box_row - dr, box_col - dc)
                ahead = (box_row + dr, box_col + dc)
                if behind not in reachable or behind in box_origins:
                    continue
                if ahead in walls or ahead in boxes:
                    continue

                new_boxes = set(boxes)
                new_boxes.remove((box_row, box_col))
                new_boxes.add(ahead)
                new_player = (box_row, box_col)
                state = (new_player, tuple(sorted(new_boxes)))
                if state in seen:
                    continue
                seen.add(state)
                action = reachable[behind] + key
                queue.append((new_player, frozenset(new_boxes), moves + action))
    raise ValueError("本局无解")
```

为了进一步压缩状态，可以在入队前排除“非目标角落”的箱子：若 `ahead` 不是目标，且其上下或左右两面组成墙角，则该状态必死。随机生成器也可能直接产生无解箱位；这时应断开并重新连接，而不是在 30 秒总时限内无限搜索。每局收到地图后求解并发送动作串，服务输出 `Yeah Continue!` 后再发送 `Y` 进入下一局。五局全部完成即打印 flag。

## 方法总结

本题先用数值的素性隐藏地图语义，再用随机推箱子迫使选手自动化。可靠解法应把“底层地形、目标、初始箱位、当前箱子、玩家”分开建模，并以推箱子而不是单步行走作为 BFS 边；每次推之前用 flood fill 取得玩家可达区和具体步行路径。标准 Sokoban 求解器若不加入“玩家不能踩目标”和“不能从初始箱位发起推动”两条题目特有限制，会生成看似正确却被服务判错的路径。最终还要实际发送五组动作并观察每局成功回包。
