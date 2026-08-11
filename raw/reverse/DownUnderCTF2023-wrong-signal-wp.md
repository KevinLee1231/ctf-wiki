# DownUnderCTF 2023 Wrong Signal Writeup

## 题目简述

程序读取 16 字节口令，把每个字节与固定 `mangle_buf` 异或，再将结果拆成四组 2 bit 方向值。程序从固定虚拟地址出发，每走一步便解引用新地址；走入不可读页会触发 `SIGSEGV`，信号处理器随即输出 `Wrong!`。完成 64 步且到达指定终点才算成功。

这些地址页实际上编码了一座 21 列宽的迷宫。本题需要先从 ELF 段权限还原可行路径，再把方向序列反编码为输入字节。

## 解题过程

构建脚本把如下迷宫逐格映射到从 `0x13370000` 开始的一系列 0x1000 字节页面：

```text
#####################
#s    #            e#
##### ######## ######
#         #         #
######### #### ######
#          #        #
####### ########## ##
#        #      #   #
# ######### ### ### #
#             #     #
#####################
```

空格、`s` 和 `e` 对应可读的 ELF `PT_LOAD` 段，墙对应没有读权限的页面。起点是 `0x13386000`，终点是 `0x13398000`。由于每行 21 格，四个方向与地址增量的关系为：

```text
0 -> -21 * 0x1000   向上
1 ->  -1 * 0x1000   向左
2 ->  +1 * 0x1000   向右
3 -> +21 * 0x1000   向下
```

把可读页看作图节点，用 BFS 从 `s` 搜索到 `e`，可得到恰好 64 步的方向数组。每四个方向按低位在前打包成一个字节，再与相同位置的混淆字节异或：

```python
directions = [-21, -1, 1, 21]
queue = [(maze.index("s"), [])]
seen = set()

while queue:
    pos, path = queue.pop(0)
    if pos in seen:
        continue
    if maze[pos] == "e":
        solution = path
        break
    for index, delta in enumerate(directions):
        nxt = pos + delta
        if 0 <= nxt < len(maze) and maze[nxt] != "#":
            queue.append((nxt, path + [index]))
    seen.add(pos)

mangle = bytes.fromhex("c2 ea 96 b6 0c 9c 92 e5 72 ff e9 3d 11 54 c1 9f")
password = bytearray()
for byte_index in range(16):
    packed = 0
    for bit_pair in range(4):
        packed |= solution[byte_index * 4 + bit_pair] << (bit_pair * 2)
    password.append(packed ^ mangle[byte_index])

print(password.decode())
```

求出的 16 字节口令为：

```text
hElCYi8OxUF7PAA5
```

题面要求把口令包在 flag 格式中，因此答案是：

```text
DUCTF{hElCYi8OxUF7PAA5}
```

## 方法总结

这里的 `SIGSEGV` 不是普通崩溃，而是迷宫碰壁判定；ELF 页权限就是地图数据。识别 0x1000 的固定步长和 21 倍步长后，可把地址运算还原成四方向网格搜索。最后按源码中的 2 bit 顺序逆打包并异或，即可从路径得到口令。
