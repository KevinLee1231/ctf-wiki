# Launch Sequence

## 题目简述

附件 `main.agc` 是 Apollo Guidance Computer（AGC Block 2）汇编程序。程序借助 DSKY 键盘与数字显示器实现了一个 $5\times6$ 的简化俄罗斯方块：玩家必须在固定的 30 个方块序列中到达指定棋盘，倒计时后屏幕才会输出组成 flag 的数字。

AGC 使用 15 位反码字，前三个寄存器为 16 位；本题还用到了 DSKY MMIO、中断和计时器。官方可用的 [yaAGC 模拟器下载页](https://www.ibiblio.org/apollo/download.html) 提供 Block 2 工具链，[AGC 汇编语言手册](https://www.ibiblio.org/apollo/assembly_language_manual.html) 则解释指令、寄存器、定点数和存储格式。下文已经给出解题所需的关键语义，无需依赖外链理解攻击过程。

## 解题过程

先还原程序主循环和经过混淆的标签，可确定棋盘宽 5、高 6；DSKY 的数字 `6`、`9`、`8` 被当作两像素组合显示。游戏结束时程序检查目标棋盘：

```text
-##-#
-####
##-##
##-#-
##-##
#-#--
```

方块种类和先后顺序由程序中的 RNG 调用次数固定。把状态定义为 `(step, board)`，枚举当前方块的所有合法旋转和横坐标，落到底部后消除满行，再用 BFS/动态规划保存前驱即可。核心转移如下：

```python
from collections import deque

queue = deque([(0, empty_board)])
parent = {}

while queue:
    step, board = queue.popleft()
    if step == len(piece_order) and board == target:
        goal = (step, board)
        break

    piece = piece_order[step]
    for rotation, shape in enumerate(piece.rotations):
        for x in range(5):
            new_board = drop_and_clear(board, shape, x)
            if new_board is None:
                continue
            state = (step + 1, new_board)
            if state not in parent:
                parent[state] = ((step, board), (piece.name, rotation, x))
                queue.append(state)
```

按照前驱反向恢复出一条有效的 30 步路径：

```text
O:0,3 T:2,0 J:2,1 T:0,1 L:0,0 O:0,3 S:1,0 S:1,3
S:1,1 I:1,4 O:0,2 T:2,0 T:1,0 J:2,2 L:2,3 L:0,0
O:0,0 J:0,2 L:3,0 I:0,1 I:0,1 O:0,0 L:1,2 Z:0,1
T:2,0 O:0,3 Z:1,3 I:0,0 L:1,2 J:3,1
```

每项格式为 `方块:旋转编号,横坐标`。实际操作时可以在模拟器中禁用由 Timer 4 驱动的重力，并把一个键盘中断映射成快速下落，避免输入速度影响结果。

目标棋盘通过校验后，倒计时结束依次显示：

```text
10646 24130 45462 77777 36416
32220 41542 44040 45022 35062
```

程序把每次移动编码为 5 位 `RRXXX`，其中高 2 位是旋转、低 3 位是横坐标。每 3 次移动正好打包成一个 15 位 AGC 字，再以 5 位八进制数显示；因此 30 次移动对应上面的 10 组数字。将屏幕显示的十组数字去掉空格，再按题目格式包裹，得到：

```text
uiuctf{10646241304546277777364163222041542440404502235062}
```

题目也接受尾部补两个 `00000` 的版本，这是未使用槽位的零填充，并不改变有效移动序列。

## 方法总结

- 核心技巧：从 AGC 汇编还原棋盘状态机和固定方块序列，再把目标状态搜索问题交给 BFS。
- 识别信号：显示逻辑围绕 DSKY 数字、棋盘更新后执行满行消除、结束时比较固定二维图案，说明输入并不是普通数字口令。
- 复用要点：模拟老式架构时必须实现反码、字长和 MMIO 语义；搜索状态应去重，并保留前驱以输出可实际复现的输入序列。
