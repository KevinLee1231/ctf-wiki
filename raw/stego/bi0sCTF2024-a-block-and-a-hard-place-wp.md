# bi0sCTF 2024 - A Block and a Hard Place

## 题目简述

服务把动态 flag 编码成一个 $37\times37$ 的二维码矩阵，再将矩阵当作二值迷宫。玩家从随机位置出发，只能根据服务返回的“同色移动成功”“跨色跳跃成功”或“无法移动”判断相邻格的颜色。目标不是寻找一条出口路径，而是在 2000 次操作内测绘完整矩阵并扫描二维码。

这道题按仓库目录位于 Misc，但决定性障碍是利用移动反馈恢复被隐藏的二维视觉载荷，因此归入 stego。

## 解题过程

### 理解两组移动指令

服务端用 `state` 记录当前格的二值状态。小写 `wasd` 只允许移动到同状态的相邻格，大写 `WASD` 只允许移动到不同状态的相邻格：

```python
if move == "d":
    if posx == width - 1 or MAZE[posy][posx + 1] != state:
        return 2
    posx += 1
    return 0

if move == "D":
    if posx == width - 1 or MAZE[posy][posx + 1] == state:
        return 2
    posx += 1
    state = MAZE[posy][posx]
    return 1
```

返回值在交互层对应为：

- `Moved!`：小写移动成功，目标格与当前格同色；
- `You can't move there!`：这次移动不满足条件，或者已经碰到边界；
- `Jumped over a wall!`：大写移动成功，目标格与当前格异色，移动后当前状态翻转。

因此，对于任意未越界的方向，先尝试小写指令；若失败，再发送对应的大写指令。前者成功说明相邻格状态不变，后者成功说明状态异或 1。每完成一次移动，就得到了一个新像素。

### 先定位左上角

起点随机，无法直接给矩阵坐标。可以反复向上探测，直到小写 `w` 与大写 `W` 都失败；两种状态都不能前进只可能是到达上边界。再用同样方法反复向左探测，即可到达左上角。

官方 solver 的注释把这里写成“top-right”，但实际指令顺序是先 `w/W`、再 `a/A`，所以落点是左上角。二维码外围静区为白色，故可把左上角初始状态设为 0；也可以在重建后用定位图案和扫码结果交叉校验这一约定。

### 蛇形扫描整个矩阵

从左上角开始逐行蛇形遍历：偶数行向右，奇数行向左，每行结束后向下移动一格。核心逻辑如下：

```python
state = 0

def step(direction):
    global state
    result = interact(direction)
    if result == 0:              # Moved!
        return state
    assert result == 2           # lowercase probe failed
    assert interact(direction.upper()) == 1
    state ^= 1
    return state
```

每个水平相邻格需要一次成功操作；只有颜色变化时额外消耗一次失败探测，换行同理。对任意 $37\times37$ 二值矩阵，极端棋盘格的理论最坏值会超过 2000 次；题目能工作依赖二维码含静区、定位图案和连续同色块，而不是一般矩阵的最坏情况保证。脚本应实时统计剩余次数，并避免重复探测已经确定的边。记录时还要统一“移动前写当前格”还是“移动后写目标格”的语义，并补上每行端点，否则很容易留下未赋值的一列。一个不易错的写法是先记录左上角，再在每次成功移动后记录新坐标：

```python
matrix = [[None] * 37 for _ in range(37)]
x = y = 0
matrix[y][x] = state

for y in range(37):
    direction = "d" if y % 2 == 0 else "a"
    dx = 1 if direction == "d" else -1
    for _ in range(36):
        step(direction)
        x += dx
        matrix[y][x] = state
    if y != 36:
        step("s")
        matrix[y + 1][x] = state
```

### 渲染并扫描二维码

将 1 渲染为黑块、0 渲染为白块，保持方形像素且不要插入额外边框。服务端在生成二维码时会不断给 flag 追加 `a`，直到矩阵尺寸达到 37；这些尾随字符是编码数据的一部分，不影响 flag 前缀和闭合花括号的识别。扫码后截取 `bi0sctf{...}` 即为动态 flag。

二维码是运行时从远端反馈重建的结果，不是仓库中的固定证据图片，因此无需为本 WP 保存一张无法复用的静态截图。

## 方法总结

本题把二维码像素伪装成可交互迷宫。小写移动是“相邻像素相同”的 oracle，大写跳跃是“相邻像素不同”的 oracle；两种操作都失败则可确认边界。先利用边界定位左上角，再蛇形遍历并随状态变化记录所有 $37\times37$ 个像素，最后渲染和扫码即可。实现时最需要检查的是坐标方向、移动前后写入时机以及每行端点是否完整。
