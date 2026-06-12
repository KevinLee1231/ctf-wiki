# 看不懂的华容道

## 题目简述

题面给出一个华容道程序，说明 flag 内容为最短路径终点对应的节点值，操作路径按棋子编号从小到大，操作顺序为 `wasd`。附件包含 `huarongdao.exe` 和 `game.bin`；程序外层有 VMP，游戏反馈能反映棋子移动和碰撞状态。题目本身要求恢复初始棋盘并找到最短合法路径，而不是普通字符串校验。

## 解题过程

VMP 保护的是一个华容道游戏逻辑，不一定要完整还原 VM。最直接的做法是在初始状态下反复尝试移动，通过碰撞反馈识别每个棋子在四个方向上是否可移动；这些反馈足够约束棋盘初始状态。

得到棋子尺寸、棋盘边界和“某棋子某方向能否移动”的指纹后，用 z3 求出初始棋盘，再对状态空间做 BFS 找最短路径。最后按题面要求把最短路径终点对应的节点值作为 flag 内容。

```python
from z3 import *
import sys
fingerprints = {
    0: [],
    1: [],
    2: [],
    3: [],
    4: ['s'],
    5: [],
    6: [],
    7: ['d'],
    8: ['s'],
    9: ['a'],
}
def solve():
    solver = Solver()
    # 1. 定义坐标变量 (x, y)
    # 棋盘范围 x:[0,3], y:[0,4]
    X = [Int(f'x_{i}') for i in range(10)]
    Y = [Int(f'y_{i}') for i in range(10)]
    # 2. 定义形状变量 (w, h)
    W = [Int(f'w_{i}') for i in range(10)]
    H = [Int(f'h_{i}') for i in range(10)]
    # ID 0: 曹操 (2x2)
    solver.add(W[0] == 2, H[0] == 2)
    # ID 6-9: 兵 (1x1)
    for i in range(6, 10):
        solver.add(W[i] == 1, H[i] == 1)
    # ID 1-5: 将 (1x2 或 2x1)
    # 假设标准配置：1个横将(2x1)，4个竖将(1x2)
    horizontal_generals = 0
    vertical_generals = 0
    for i in range(1, 6):
        # 宽只能是 1 或 2
        solver.add(Or(W[i] == 1, W[i] == 2))
        # 如果宽是 1，高就是 2；如果宽是 2，高就是 1
        solver.add(H[i] == 3 - W[i])
    # 约束：ID 1-5 中，宽度的总和必须是 (1*4 + 2*1) = 6
    # 这确保了恰好有 1 个横将，4 个竖将
    solver.add(Sum([W[i] for i in range(1, 6)]) == 6)
    for i in range(10):
        # 边界约束
        solver.add(X[i] >= 0, Y[i] >= 0)
        solver.add(X[i] + W[i] <= 4)
        solver.add(Y[i] + H[i] <= 5)
        # 不重叠约束
        for j in range(i + 1, 10):
            # i 在 j 的左、右、上、下
            solver.add(Or(
                X[i] + W[i] <= X[j], # i 在 j 左边
                X[j] + W[j] <= X[i], # j 在 i 左边
                Y[i] + H[i] <= Y[j], # i 在 j 上边
                Y[j] + H[j] <= Y[i]  # j 在 i 上边
            ))
    # 定义：检查棋子 i 往 (dx, dy) 移动是否合法
    # 如果合法 -> 不被边界阻挡 AND 不被其他棋子阻挡
    def is_move_valid_expr(i, dx, dy):
        new_x = X[i] + dx
        new_y = Y[i] + dy
        # 边界阻挡条件 (Blocked if out of bounds)
        boundary_blocked = Or(
            new_x < 0,
            new_x + W[i] > 4,
            new_y < 0,
            new_y + H[i] > 5
        )
        # 棋子碰撞条件
        collision_list = []
        for j in range(10):
            if i == j: continue
            # not (i_right <= j_left or i_left >= j_right or
            #      i_bot <= j_top or i_top >= j_bot)
            overlap = Not(Or(
                new_x + W[i] <= X[j],
                X[j] + W[j] <= new_x,
                new_y + H[i] <= Y[j],
                Y[j] + H[j] <= new_y
            ))
            collision_list.append(overlap)
        return And(Not(boundary_blocked), Not(Or(collision_list)))
    for pid, moves in fingerprints.items():
        # 检查四个方向
        dirs = {'w': (0, -1), 's': (0, 1), 'a': (-1, 0), 'd': (1, 0)}
        for d_char, (dx, dy) in dirs.items():
            can_move = (d_char in moves)
            if can_move:
                solver.add(is_move_valid_expr(pid, dx, dy))
            else:
                solver.add(Not(is_move_valid_expr(pid, dx, dy)))
    # === 求解 ===
    print("正在计算初始状态...")
    if solver.check() == sat:
        m = solver.model()
        print("\n[+] 成功碰撞出初始状态！")
        # 绘制棋盘
        board = [['.' for _ in range(4)] for _ in range(5)]
        results = []
        for i in range(10):
            x = m[X[i]].as_long()
            y = m[Y[i]].as_long()
            w = m[W[i]].as_long()
            h = m[H[i]].as_long()
            results.append({'id': i, 'x': x, 'y': y, 'w': w, 'h': h})
            # 填充可视化棋盘
            for r in range(y, y+h):
                for c in range(x, x+w):
                    board[r][c] = str(i)
        for row in board:
            print(" ".join(row))
        for r in sorted(results, key=lambda k: k['id']):
            print(f"ID {r['id']}: ({r['x']}, {r['y']}) Size {r['w']}x{r['h']}")
        return results
    else:
        return None
if __name__ == '__main__':
    solve()
```

## 方法总结

遇到 VM 保护的游戏/迷宫题，不必一开始完整反编译 VM。若程序会对移动结果给出稳定反馈，可以先把反馈当 oracle，恢复状态空间和约束，再用搜索算法求路径；逆向只服务于建模所需的最小信息。
