# Mikusweeper

## 题目简述

题目是一局限时的在线扫雷寻路游戏。客户端通过 WebSocket 接收当前状态，并发送以换行分隔的 `up`、`down`、`left`、`right` 指令控制角色。地图固定为 $50\times 30$，随机放置 $300$ 至 $309$ 颗雷、$40$ 把钥匙，玩家有 $8$ 条命，必须在 $80$ 秒内收集全部钥匙。

服务端消息的核心结构为：

```json
{
  "hero": {"x": 12, "y": 8},
  "map": [["covered", "c1", "key"]],
  "numKeysRetrieved": 0,
  "livesRemaining": 8,
  "flag": "仅胜利时出现"
}
```

格子状态 `c0` 至 `c8` 表示周围八格的雷数，`covered` 表示未揭开，`bomb` 和 `key` 分别表示已经揭出的雷与钥匙。移动使用四邻接，但扫雷数字统计八邻接。收集完 $40$ 把钥匙后，服务端返回：

```text
SEKAI{M1ku_K1ngd0m_h4s_b33n_54v3d_OwO<3}
```

## 解题过程

自动求解器循环接收状态，并按“先拿现成钥匙、再走确定安全格、最后才猜”的顺序行动。

第一步是路径搜索。把已揭开的 `c0` 至 `c8` 和 `key` 视为可通行格，把 `covered` 与已知 `bomb` 视为障碍，用 BFS 求角色到目标格的最短四邻接路径。将相邻坐标差转换为方向字符串后，可以把整条路径一次发送，减少 WebSocket 往返耗时。

第二步是反复应用基础扫雷推理。对每个已揭开的数字格，记数字为 $d$：

- 若其八邻域中“未揭格 + 已标雷”的数量恰好为 $d$，则所有未揭格都可标为雷。
- 若其八邻域中已标雷的数量恰好为 $d$，则其余未揭格都是安全格。

对应的核心逻辑可写成：

```python
def infer_bombs(board):
    changed = True
    while changed:
        changed = False
        for y, row in enumerate(board):
            for x, state in enumerate(row):
                if not state.startswith("c") or state == "c0":
                    continue
                number = int(state[1])
                unknown = [
                    point for point in around8(y, x)
                    if board[point[0]][point[1]] == "covered"
                ]
                known_bombs = sum(
                    board[ny][nx] == "bomb"
                    for ny, nx in around8(y, x)
                )
                if known_bombs + len(unknown) == number:
                    for ny, nx in unknown:
                        board[ny][nx] = "bomb"
                    changed |= bool(unknown)
    return board

def is_safe(board, y, x):
    if board[y][x] != "covered":
        return False
    for ny, nx in around8(y, x):
        state = board[ny][nx]
        if state.startswith("c"):
            number = int(state[1])
            known_bombs = sum(
                board[ay][ax] == "bomb"
                for ay, ax in around8(ny, nx)
            )
            if known_bombs == number:
                return True
    return False
```

收集和移动策略如下：

1. 扫描地图中的可见 `key`，若 BFS 可达，就走最短路拾取。
2. 没有可见钥匙时，先标出所有能确定的雷，再找所有能确定安全的 `covered` 格。
3. 按与当前位置的曼哈顿距离排序安全格；临时把目标格当作可通行格，逐个用 BFS 串成一批移动指令。
4. 若基础规则无法推出任何安全格，选择最近且可达的未揭格猜测，依靠 $8$ 条命容忍少量失误。

官方脚本还说明了一个失败边界：错误猜测可能把角色困进由雷围成的连通块，而脚本没有实现“主动踩雷脱困”。因此它是概率型求解器，通常数次重连即可在时限内成功，不能把单次失败误判为算法或协议错误。

## 方法总结

本题把扫雷约束与网格寻路组合在一起：八邻接规则用于推理雷和安全格，四邻接 BFS 用于实际移动。批量发送路径解决时限问题，生命值则为无法从局部数字唯一推断时的随机猜测提供容错。最容易出错的是混淆两种邻接关系，或让 BFS 穿过仍为 `covered` 的未知格。
