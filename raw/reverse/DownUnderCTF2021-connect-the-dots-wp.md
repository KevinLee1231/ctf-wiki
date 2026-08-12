# DownUnderCTF 2021 - connect the dots

## 题目简述

程序接收一串 `h/j/k/l/x` 指令，在 $60\times60$ 迷宫中移动。八个点必须各访问一次，且访问顺序需要使内部状态最终等于 `0xff`；随后程序又把整条移动串按 81 字节分块，生成 44 字节密钥并解密 flag。因此不仅要找到正确的点序，还要复现确定的迷宫路径。

## 解题过程

### 解析迷宫字节

`MAZE_DATA` 每个元素编码一个格子。低四位分别表示四个方向的墙：

| 指令 | 位掩码 | 位置变化 |
| --- | ---: | ---: |
| `h` | `0x01` | $-1$ |
| `j` | `0x02` | $+60$ |
| `l` | `0x04` | $+1$ |
| `k` | `0x08` | $-60$ |

最高位 `0x80` 表示该格有点，位 4 至位 6 保存点编号：

```python
if cell & 0x80:
    dot_idx = (cell >> 4) & 7
```

从左上角位置 0 开始，八个点的坐标为：

```text
(15,11), (35,20), (18,24), (54,28),
(11,35), (34,39), (57,48), (45,59)
```

### 枚举正确的点序

在点上执行 `x` 后，程序拒绝重复点，并按该点对应的 `DOTS_DATA` 更新状态：

```c
win_bits &= DOTS_DATA[dot_idx] >> 8;
win_bits ^= DOTS_DATA[dot_idx] & 0xff;
```

八个点只有 $8!=40320$ 种顺序，可以直接穷举：

```python
from itertools import permutations

def find_order(values):
    for order in permutations(range(8)):
        state = 0
        for idx in order:
            z = values[idx]
            state &= z >> 8
            state ^= z & 0xff
        if state == 0xff:
            return order
```

得到点编号顺序：

```text
6, 3, 7, 5, 1, 4, 0, 2
```

### 用 BFS 连接八个检查点

对每一对相邻检查点运行 BFS。扩展某个格子时，只在当前格对应墙位为零时加入邻居，并记录产生该边的移动字符：

```python
MOVES = [
    (-1,  "h", 0x01),
    (60,  "j", 0x02),
    (1,   "l", 0x04),
    (-60, "k", 0x08),
]

def neighbours(maze, pos):
    for delta, key, wall in MOVES:
        nxt = pos + delta
        if 0 <= nxt < len(maze) and not (maze[pos] & wall):
            yield nxt, key
```

从位置 0 到顺序中的第一个点，再逐点搜索；每到一个点就在路径后追加 `x`。本地运行官方 solver 得到的获胜串长度为 3566 字节。完整机械路径由 BFS 根据 `maze_data` 确定，保留生成算法比在题解中粘贴数千字符更易复现。

### 由移动串解密 flag

程序在迷宫判定成功后，以连续 81 个移动字符的乘积模 255 作为一个密钥字节：

$$
k_i=\prod_{j=0}^{80}\operatorname{ord}(moves_{81i+j})\bmod255.
$$

再与内置密文逐字节异或：

```python
def product_mod_255(block):
    value = 1
    for c in block:
        value = value * c % 255
    return value

key = [
    product_mod_255(win_moves[i * 81:(i + 1) * 81].encode())
    for i in range(44)
]
flag = bytes(c ^ k for c, k in zip(CT, key))
```

3566 字节的路径足以覆盖 $44\times81=3564$ 个参与解密的字符，最后两个字符不进入密钥计算。输出为：

```text
DUCTF{bfs_dfs_ffs_no_more_mazes_a8fb66c12cd}
```

## 方法总结

本题包含两层约束：先枚举八个点的状态更新顺序，再在带墙位掩码的迷宫上用 BFS 连接这些点。程序最后把移动路径本身用作解密材料，所以不能只证明“存在一条通路”；必须生成与 solver 一致、长度足够的实际按键串。穷举 $8!$ 个点序和八次 BFS 都很小，没必要对整个路径空间做联合暴力搜索。
