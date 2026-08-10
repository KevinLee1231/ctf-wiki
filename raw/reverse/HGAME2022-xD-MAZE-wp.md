# xD MAZE

## 题目简述

程序把一张四维 $8\times8\times8\times8$ 迷宫展平成长度为 $8^4=4096$ 的一维数组。输入是 28 个数字，每个数字选择一个正方向；只有每一步落到空格字符时才通过，最终输入会被包成 `hgame{...}`。

## 解题过程

设四维坐标为 $(a,b,c,d)$，展平下标为：

$$
\operatorname{index}(a,b,c,d)=8^3a+8^2b+8c+d
$$

反编译代码中的四个输入数字与下标增量对应如下：

| 输入 | 坐标变化 | 一维增量 |
| --- | --- | ---: |
| `3` | $d\leftarrow d+1$ | $1$ |
| `2` | $c\leftarrow c+1$ | $8$ |
| `1` | $b\leftarrow b+1$ | $64$ |
| `0` | $a\leftarrow a+1$ | $512$ |

在 IDA 中把 4096 字节的迷宫数组导出为 `maze.bin`，然后对四种正向移动做深度优先搜索。下面的脚本会寻找恰好 28 步、且每一步均落在空格 `0x20` 上的路径：

```python
from functools import lru_cache
from pathlib import Path

SIZE = 8
STEPS = 28
maze = Path("maze.bin").read_bytes()

if len(maze) != SIZE ** 4:
    raise ValueError(f"迷宫应为 {SIZE ** 4} 字节，实际为 {len(maze)} 字节")

# 按官方程序的判断顺序搜索。
moves = (
    (1, "3"),
    (SIZE, "2"),
    (SIZE ** 2, "1"),
    (SIZE ** 3, "0"),
)

@lru_cache(maxsize=None)
def search(position: int, depth: int):
    if depth == STEPS:
        return ""

    for delta, digit in moves:
        next_position = position + delta
        if next_position < len(maze) and maze[next_position] == 0x20:
            suffix = search(next_position, depth + 1)
            if suffix is not None:
                return digit + suffix
    return None

path = search(0, 0)
if path is None:
    raise RuntimeError("未找到 28 步通路，请检查数组起点和导出范围")

print(f"hgame{{{path}}}")
```

官方 PDF 只展示了依赖外部 `data.py` 的简化脚本，没有保存 4096 字节数组和最终输出。参赛者复现文档给出了同一迷宫的搜索结果，并可与上述方向映射逐位核对：[HGAME2022 Reverse Writeup](https://secondbc.github.io/SecondBC/2022/02/22/Hgame2022-ReverseWriteUp/)。最终 flag 为：

```text
hgame{3120113031203203222231003011}
```

## 方法总结

这题的关键不是识别某种编码，而是把一维访问偏移还原成四维网格：$1,8,64,512$ 正好是 $8^0,8^1,8^2,8^3$。面对大常量数组时，不必把 4096 个字符贴进题解；应说明数组如何从二进制导出，并给出能直接读取导出物的完整搜索器。这样既保留复现路径，也避免用没有视觉价值的巨型迷宫截图占据正文。
