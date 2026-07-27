# Forest

## 题目简述

这是一道基于 Windows 异常处理的迷宫题。程序用 `VirtualProtect` 给内置代码区增加执行权限，把输入转换成二进制位，再执行 `int 3` 进入位于 `sub_401A00` 的 SEH 处理逻辑。

第一次 `DEBUGGER_EXCEPTION` 会为当前位置 $(x,y)$ 和当前 flag bit 分配状态，并把 shellcode 中的 `0xffffffff` 占位地址替换为这些状态变量的真实地址。处理器随后在异常上下文中设置 Trap Flag：

```c
context->EFlags |= 0x100;
```

CPU 每执行一条指令都会产生 `SINGLE_STEP_EXCEPTION`。SEH 根据当前输入 bit 选择迷宫块的两个后继之一；走到退出块会触发访问异常并成功结束，走入 `int 3` 或自环块则失败。预期解法是不实际执行迷宫，而是静态解析每个块的两个目标，构建有向图并寻找从起点到出口的路径。

## 解题过程

### 识别 17×17 的块布局

迷宫位于 `forest.exe` 文件偏移 `0x4a30`，总长度为 `0x4840`：

$$
\frac{\mathtt{0x4840}}{\mathtt{0x40}}
=289
=17\times17
$$

因此共有 $17\times17$ 个块，每块固定 `0x40` 字节。坐标 $(y,x)$ 对应的相对偏移为：

$$
\mathrm{offset}=0x40\times(17y+x)
$$

出口块的相对偏移是 `0x6c0`。

### 区分三类块

普通检查块以如下字节开头：

```text
b8 ff ff ff ff 8b 00
```

即先把待修补的绝对地址放进 `eax`，再读取当前 flag bit。后面连续出现两组目标坐标写入：

```text
b8 ff ff ff ff c7 00 <x:4>
b8 ff ff ff ff c7 00 <y:4>
fa
```

`0xfa` 是特权指令 `cli`，用户态执行会产生异常，SEH 用它标记一次分支结束。第一组坐标对应 bit 0，第二组坐标对应 bit 1。

另外两类块为：

- `cd 03`：再次触发调试异常。该异常只应在初始化时出现一次，二次触发表示走入死路或受到调试器影响；
- 两个后继都指向自身的检查块：程序逐 bit 校验时会永久停在原地，用于阻止直接枚举；
- 相对偏移 `0x6c0`：胜利出口。

### 构图并恢复路径

把每个普通块视为一个顶点，并添加两条带标签的边：

```text
当前块 --0--> left
当前块 --1--> right
```

从 `(0, 0)` 做广度优先搜索即可得到通往出口的最短路径。沿父节点反向恢复时，同时收集每条边的 0/1 标签；每 8 bit 按高位在前转换为字符，就是 flag 内容。

下面的脚本只依赖 Python 标准库，不需要原题解中的 `networkx`：

```python
#!/usr/bin/env python3
from collections import deque
from dataclasses import dataclass
from pathlib import Path


FILE_OFFSET = 0x4A30
MAZE_SIZE = 0x4840
BLOCK_SIZE = 0x40
WIDTH = 17
WIN_OFFSET = 0x6C0

LOAD_FLAG_BIT = b"\xb8\xff\xff\xff\xff\x8b\x00"
WRITE_DWORD = b"\xb8\xff\xff\xff\xff\xc7\x00"


@dataclass(frozen=True)
class Block:
    kind: str
    zero: tuple[int, int] | None = None
    one: tuple[int, int] | None = None


image = Path("forest.exe").read_bytes()
maze = image[FILE_OFFSET:FILE_OFFSET + MAZE_SIZE]
if len(maze) != MAZE_SIZE:
    raise ValueError("forest.exe is shorter than expected")


def read_assignment(cursor: int) -> tuple[int, int]:
    if maze[cursor:cursor + 7] != WRITE_DWORD:
        raise ValueError(f"bad assignment at {cursor:#x}")
    value = int.from_bytes(
        maze[cursor + 7:cursor + 11],
        "little",
    )
    return value, cursor + 11


def get_block(position: tuple[int, int]) -> Block:
    y, x = position
    if not (0 <= y < WIDTH and 0 <= x < WIDTH):
        raise ValueError(f"out-of-range position: {position}")

    offset = BLOCK_SIZE * (WIDTH * y + x)
    if offset == WIN_OFFSET:
        return Block("win")
    if maze[offset:offset + 2] == b"\xcd\x03":
        return Block("fail")
    if maze[offset:offset + 7] != LOAD_FLAG_BIT:
        raise ValueError(f"unknown block at {position}, {offset:#x}")

    cursor = offset + 7

    zero_x, cursor = read_assignment(cursor)
    zero_y, cursor = read_assignment(cursor)
    if maze[cursor] != 0xFA:
        raise ValueError(f"missing first cli at {cursor:#x}")
    cursor += 1

    one_x, cursor = read_assignment(cursor)
    one_y, cursor = read_assignment(cursor)
    if maze[cursor] != 0xFA:
        raise ValueError(f"missing second cli at {cursor:#x}")

    return Block(
        "branch",
        zero=(zero_y, zero_x),
        one=(one_y, one_x),
    )


start = (0, 0)
queue = deque([start])
parent: dict[
    tuple[int, int],
    tuple[tuple[int, int] | None, int | None],
] = {start: (None, None)}
end = None

while queue:
    current = queue.popleft()
    block = get_block(current)

    if block.kind == "win":
        end = current
        break
    if block.kind == "fail":
        continue

    for bit, target in enumerate((block.zero, block.one)):
        assert target is not None
        if target not in parent:
            parent[target] = (current, bit)
            queue.append(target)

if end is None:
    raise RuntimeError("no path to the win block")

bits = []
current = end
while parent[current][0] is not None:
    previous, bit = parent[current]
    assert previous is not None and bit is not None
    bits.append(str(bit))
    current = previous
bits.reverse()

bit_string = "".join(bits)
if len(bit_string) % 8 != 0:
    raise ValueError(f"bit length is not byte-aligned: {len(bit_string)}")

body = "".join(
    chr(int(bit_string[index:index + 8], 2))
    for index in range(0, len(bit_string), 8)
)

print(bit_string)
print(f"d3ctf{{{body}}}")
```

输出：

```text
d3ctf{0ut_of_th3ForesT#}
```

## 方法总结

本题通过 `DEBUGGER_EXCEPTION`、单步异常和特权指令异常把普通分支隐藏进 SEH 状态机。动态单步会反复进入异常处理器，还可能触发“调试异常只能出现一次”的反调试分支；静态解析固定尺寸块更稳定。

关键抽象是把每个块还原为“当前坐标、bit 0 后继、bit 1 后继”。一旦得到这个带标签有向图，异常机制就不再影响求解，flag 等于从起点到出口路径上的边标签。自环块也无需特殊爆破：BFS 的 visited 集合会自然忽略它。
