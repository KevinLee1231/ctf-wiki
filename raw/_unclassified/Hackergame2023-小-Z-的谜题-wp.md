# 小 Z 的谜题

## 题目简述

题目要求在坐标范围 $[0,5]^3$ 内放置 16 个可旋转的轴对齐长方体。六种尺寸及数量为

$$
(1,1,3)^3, (1,2,2)^4, (1,2,4)^2, (1,4,4)^2,
(2,2,2)^2, (2,2,3)^3.
$$

每块用三个维度的起止坐标表示，最终将全部 96 个十进制数位连续提交。校验器要求列表按坐标字典序排序、任意两块至少在一个轴上不相交，并恰好使用规定数量的各类积木。

得分不是占用体积，而是对每块的三组 `[(low, high, -1)]` 做笛卡尔积，再统计所有三元组去重后的数量；`score <= 136`、`137..156`、`score >= 157` 分别对应三档结果。决定性主障碍是三维组合装箱、精确覆盖和约束搜索，没有可稳定映射的安全方向，因此归入 `_unclassified`。

## 解题过程

### 还原校验条件

每块长方体 $i$ 的坐标写作

$$
([x_i^0,x_i^1],[y_i^0,y_i^1],[z_i^0,z_i^1]).
$$

两个长方体 $i,j$ 不重叠当且仅当至少存在一个轴 $k$ 满足

$$
x_{i,k}^1\le x_{j,k}^0
\quad\text{或}\quad
x_{j,k}^1\le x_{i,k}^0.
$$

尺寸匹配时对三条边长排序，因此允许旋转。排序约束无需放入搜索模型：得到所有摆放后对 16 个坐标三元组排序，再序列化即可。

### 低分解与高分解的不同搜索方向

中间档只需任意合法构造。低分档要求许多端点和投影坐标重合，可用 DFS 优先把若干小块组合成规则大块，减少候选摆放和 score；每次加入候选时检查边界、尺寸计数、与已放积木不相交，并对当前 score 做剪枝。

高分档不能随意合并。所有积木体积之和恰好为

$$
3\cdot3+4\cdot4+2\cdot8+2\cdot16+2\cdot8+3\cdot12=125=5^3,
$$

所以“全部在盒内且互不重叠”等价于完整铺满 125 个单位格，可以自然转成精确覆盖。

### 用 Algorithm X / DLX 建模

精确覆盖的列分两组：

- 16 个“具体积木被使用一次”列；同尺寸积木仍按编号区分；
- 125 个“单位格被覆盖一次”列。

对每块积木枚举边长的所有不同排列以及所有不越界起点。一个候选摆放构成一行，覆盖该积木编号列和它占据的所有单位格列：

```python
import itertools
import math

blocks = ((1, 1, 3),) * 3 + ((1, 2, 2),) * 4 \
       + ((1, 2, 4),) * 2 + ((1, 4, 4),) * 2 \
       + ((2, 2, 2),) * 2 + ((2, 2, 3),) * 3
target = (5, 5, 5)
assert sum(math.prod(b) for b in blocks) == math.prod(target)

rows = []
placement = []
for block_id, size in enumerate(blocks):
    for sx, sy, sz in set(itertools.permutations(size)):
        for x0 in range(6 - sx):
            for y0 in range(6 - sy):
                for z0 in range(6 - sz):
                    cells = []
                    for x in range(x0, x0 + sx):
                        for y in range(y0, y0 + sy):
                            for z in range(z0, z0 + sz):
                                cells.append(16 + x * 25 + y * 5 + z)
                    rows.append([block_id, *cells])
                    placement.append(((x0, x0 + sx),
                                      (y0, y0 + sy),
                                      (z0, z0 + sz)))
```

将这些行交给 Algorithm X 或 SageMath `DLXMatrix`，每个解正好选择 16 行。官方实现还对体积为 3 的长条加入 parity 列：记录它沿 $x,y,z$ 三轴覆盖的坐标层集合，把 Conway puzzle 的奇偶约束一并编码进精确覆盖，可显著缩小搜索树。

对每个 DLX 解，取对应 `placement`、字典序排序，并严格复刻服务端计算：

```python
score = len({
    (x, y, z)
    for box in arrange
    for x, y, z in itertools.product(
        [box[0][0], box[0][1], -1],
        [box[1][0], box[1][1], -1],
        [box[2][0], box[2][1], -1],
    )
})

payload = "".join(
    str(box[axis][end])
    for box in sorted(arrange)
    for axis in range(3)
    for end in range(2)
)
```

提交前应重新断言 payload 恰有 96 个字符、每个字符在 `0..5`，并用原校验器检查无重叠、尺寸计数和目标 score 区间。官方 DLX 示例运行约数分钟，Z3 高分模型可能需要半小时；本轮按禁止长时运行要求只静态核对模型，没有重新搜索。

## 方法总结

- 核心技巧：利用总体积等于容器体积，把三维装箱转为“积木编号列 + 单位格列”的精确覆盖，再用 DLX 枚举并按原程序评分。
- 识别信号：有限种积木的全部摆放可枚举、要求每块恰用一次且空间恰好铺满时，应优先考虑 Algorithm X，而不是直接对所有坐标做无结构暴力。
- 复用要点：先精确复制评分函数，不能把“端点与投影去重数”误当体积；对称、排序和 parity 约束可以在不改变解集的前提下大幅剪枝。
