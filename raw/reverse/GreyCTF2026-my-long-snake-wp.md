# My Long Snake

## 题目简述

题目在 GreyFlag 徽章上运行一个确定性 Snake 游戏。选手把只包含 `U/D/L/R` 的动作写入 `/moves.txt`，程序依次重放；初始蛇为 `(9,9),(8,9),(7,9)`、方向向右，需要在最多 4096 步内吃到 40 个苹果，且不能立即反向、撞墙或撞到自身。

虽然题目目录位于 Hardware，但关键障碍是从公开游戏代码还原状态机并生成有效输入，不依赖总线、射频或器件状态，因此按 Reverse 归档。

## 解题过程

棋盘尺寸为 $18\times18$，外围一圈是墙，实际可走坐标为 $1\le x,y\le16$。苹果使用固定种子 `0x5A17C0DE` 的 xorshift32 生成：

```python
x ^= (x << 13) & 0xffffffff
x ^= x >> 17
x ^= (x << 5) & 0xffffffff
x &= 0xffffffff
```

每个候选苹果连续消耗两个输出：

```python
apple_x = 1 + prng.next() % 16
apple_y = 1 + prng.next() % 16
```

若候选格被当前蛇身占据，程序会继续消耗 PRNG 输出并重选。因此不能脱离蛇的实际移动轨迹预先列出全部苹果；求解器必须逐步模拟蛇身、增长状态和 PRNG。第一个苹果位于 `(9,7)`。

稳定基线是在 $16\times16$ 可玩区中沿 Hamiltonian-cycle 风格的蛇形路线移动，并只在不会破坏身体沿环顺序的情况下向当前苹果取安全捷径。每一步都按题目顺序验证：

```python
next_head = add(head, direction)
grow = next_head == food
collision_body = snake if grow else snake[:-1]

assert next_head inside playable_area
assert next_head not in collision_body

snake.insert(0, next_head)
if grow:
    food = spawn_food(prng, snake)
else:
    snake.pop()
```

把下面完整序列保存为 `/moves.txt`：

```text
DDDDDDDLLLLLLLLUUUUUUUUURRRRRRRRDDDDDDDDDLLLLLLLLUUUUUUUUUUURRRRRRRRDDDDDDDDDDDLLLLLLLLUUUURDRRRDDDLLLLUUUUUUURRRRRRRRULLLLLDDDDDDDDLLLUUUUUUUUUUURRRDDDDDDDDRRDDDLLLLLUUUUUUURRRRRRRRRRDDRDDDDDLLLLLLLLLLLUUUUUUUUUUUUUUURRRRRRRRRRDDDDDDDDDDDDDDDLLLLLLLLLLUUUUURRRRRRRRRDDDDDLLLLLLLLLUUUUUUUUURRRRRRRRRRRRDDDLLLLLLLLLDDDDDDLLLUUUUUUUUUUURRRRRRRRRRRRRRDDDDDDDDDDDLLLLLLLLLLLLLLUUUUUUUUUUUUUUURRRRRRRRRRRDDDDDDDLDDLLLDDLLLLLLLUUUUUUURRRRRRRRRRRRRDDDDDDDDDDDLLLLLLLLLLLLLURRRRRRRRRRRRRDLLLLLLLLLLLLLUUUUUUUUUUUUURRDDDDDDDDDDDDDLLUUUUUUUUUUUUUUURRRRRRRRRRDDDDDDDDDLLLLLLLLDDDRRRRRRRRRRDDDLLLLLLLLLLLLUUUUUUUUUUUUUUURRRRRRRRRRRDDDDDDDDDLLLLLDDDDDDLLLLLLUUUUUUUUUUUUURRRDDDDRRRDDRRRDDDDDDDLLLLLLLLLUUUUUUUUUUUUUUURRRRRRRRRRRRDDDDDDDDDLLLLDDDDDRRRRRRDLLLLLLLLLLLLLLUUUUUUUUUUURRRRRRRRRRRRRD
```

按源码重放验证，该文件共 780 步，第 780 步恰好吃到第 40 个苹果，最终蛇长为 43。徽章随后打印：

```text
grey{i_have_a_long_snakey}
```

## 方法总结

固定种子只保证过程可复现，并不意味着苹果坐标与玩家路径无关：占用格拒绝采样会让 PRNG 消耗量受蛇身影响。正确方法是完整镜像游戏状态，再用维持 Hamiltonian 环顺序的路线避免长蛇封死自己。提交前应使用原始碰撞规则离线重放整个 `moves.txt`，同时核对动作数、苹果数和首次失败位置。
