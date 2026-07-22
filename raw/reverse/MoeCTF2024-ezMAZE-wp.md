# ezMAZE

## 题目简述

程序把 $80\times56$ 的迷宫按位压缩为 `56 × 10` 字节：每行一个 bit 表示一个格子，1 为墙、0 为可走。玩家从 `(2, 2)` 出发到 `(75, 55)`，且 `checkForMaze` 会把访问过的格子置 1，禁止重复经过。找到路径后，程序还用完整 1176 字节路径计算 flag 中的十进制数。

## 解题过程

坐标 `(x, y)` 对应的字节和 bit 为：

$$
byte=(y-1)\times10+\left\lfloor\frac{x-1}{8}\right\rfloor,
$$

$$
bit=7-((x-1)\bmod8).
$$

从反编译结果抄出 560 字节迷宫后，用 BFS 记录父节点即可。以下脚本包含完整迷宫数据，并同时复现 flag 计算：

```python
from collections import deque

maze = bytes.fromhex("""
ffffffffffffffffffffbfffeaa8a4924fffffff80003d149492b80000ffffffbbda1f297bfffeff
c0003bdadc00030000ffdffffbdaddffff7fffffc0003bdadc00030000ffffffb8421ffffbfffeff
c0003ffffc00030000ffdffff87c3dffff7fffffc0003bfdbc0003003e1fffffb87c7ffffbffbedf
ffffbf7dbd297bffbedfffff987c3ffffbffbedf80004ffff80002003edfbf7f6ffffbfffefffedf
96ab600008000201fedf96ab7fffeffffbfdfedf954b70000c000301fedf954b77fffdffff7ffedf
aa9670000c00030006df92577fffeffffbfff6df929470000c00030006df8a4b77fffdffff7ffedf
aed570000c0003000edf954affffeffffbffeedf8a4a4a50ec0003ffeedfbf7f7fffe1ffffffeedf
fffff55d7fffffffeedf800038d5352a98000edfbfffbd512ffffbfffedfa0003ffffc00030000df
afffffc07dffff7fffdfa0003fdf7c00030000dfbfffbfdf7ffffbfffedfa0003fdf7c00030000df
afffffdf7dffff7fffdfa0003fdf7c00030000dfbfffbfdf7ffffbfffedfbc8bbfdf7ffffbfffedf
a9249fdf7ffffbfffe5fb2a1cfdf78000200035fb152efdffbfffefffb5faa22e00008000201f85f
bd29ffffeffffbfdffdfb252b0000c000301ffdfb494b7fffdffff7fffdfb24ab0000c00030000df
b2525fffeffffbfffedfb254d0000c00030000dfb25257fffdffff7fffdfb291300000000f0000df
b0897ffffffff87ffedfbc9110ffe00003fffedf800000000dfffe948a5fffffffffffffffffffff
""")

def is_open(x, y):
    if not (1 <= x <= 80 and 1 <= y <= 56):
        return False
    value = maze[(y - 1) * 10 + (x - 1) // 8]
    bit = 7 - ((x - 1) % 8)
    return ((value >> bit) & 1) == 0

start = (2, 2)
goal = (75, 55)
moves = (
    ("w", 0, -1),
    ("s", 0, 1),
    ("a", -1, 0),
    ("d", 1, 0),
)

queue = deque([start])
previous = {start: (None, "")}

while queue:
    current = queue.popleft()
    if current == goal:
        break
    x, y = current
    for char, dx, dy in moves:
        nxt = (x + dx, y + dy)
        if nxt not in previous and is_open(*nxt):
            previous[nxt] = (current, char)
            queue.append(nxt)

assert goal in previous
path = []
current = goal
while current != start:
    current, char = previous[current]
    path.append(char)
path = "".join(reversed(path))
assert len(path) == 1176

# MSVC 在这里先按 32 位有符号 int 计算每一项，再累加到 uint64_t。
total = 0
for index, char in enumerate(path.encode()):
    term = (char * 3113131 * index + index * index + 0x114514) & 0xFFFFFFFF
    if term & 0x80000000:
        term -= 1 << 32
    total = (total + term) & 0xFFFFFFFFFFFFFFFF

print(path)
print(f"moectf{{the_{total}_amazing_maze!!}}")
```

输出的路径长度正好是 `scanf_s("%1177s", ...)` 能接收的 1176 字节，最终结果为：

```text
moectf{the_18446744024826406994_amazing_maze!!}
```

若直接按 Python 无界正整数求和，会得到另一个数；必须模拟附件编译器中每一项的 32 位有符号回绕，再转入 64 位无符号累加。

## 方法总结

状态压缩迷宫的关键是正确还原 MSB-first 的 bit 坐标。路径搜索本身用普通 BFS 即可，但 flag 还依赖编译器整数宽度；逆向题里的“相同公式”若没有复现类型提升和溢出，结果仍会错误。
