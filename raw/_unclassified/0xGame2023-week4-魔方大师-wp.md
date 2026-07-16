# 魔方大师

## 题目简述

服务端实现了三阶魔方并提供三档难度。第三档会连续打乱魔方 50 次，每轮操作受 3 秒闹钟限制；只有每轮都还原并提交检查，最终才会输出 flag。需要自动解析终端真彩色方块、调用魔方求解器并转换步骤语法。

## 解题过程

### 理清服务端协议

服务端使用 ANSI `38;2;R;G;Bm` 真彩色转义序列显示 54 个面片，展开顺序为：顶部 U，随后逐行输出 L/F/R/B，最后输出 D。`kociemba` 则要求把 54 个面片排列为 URFDLB，因此不能直接把正则提取结果交给求解器。

颜色和求解器面符号的对应关系由各面的中心块确定。结合源码中的初始魔方，可以得到：

```text
黄色 -> F    白色 -> B    红色 -> R
绿色 -> D    橙色 -> L    蓝色 -> U
```

服务端接受的转动语法也与求解器不同：大写字母表示顺时针，小写字母表示逆时针。因此 `R'` 要转换成 `r`，`R2` 要转换成 `RR`。

### 自动完成 50 轮

下面的脚本保留了完整的数据重排和交互逻辑，并用占位符替代已经失效的比赛地址：

```python
import re

import kociemba
from pwn import args, remote


RGB_TO_FACE = {
    (255, 255, 0): "F",
    (255, 255, 255): "B",
    (255, 0, 0): "R",
    (0, 255, 0): "D",
    (255, 150, 50): "L",
    (0, 0, 255): "U",
}
COLOR_RE = re.compile(r"\x1b\[38;2;(\d+);(\d+);(\d+)m")


def parse_cube(text: str) -> tuple[str, str]:
    colors = [
        RGB_TO_FACE[tuple(map(int, match))]
        for match in COLOR_RE.findall(text)
    ]
    if len(colors) != 54:
        raise ValueError(f"期望 54 个面片，实际得到 {len(colors)} 个")

    up = colors[:9]
    left, front, right, back = [], [], [], []
    offset = 9
    for _ in range(3):
        left.extend(colors[offset : offset + 3])
        front.extend(colors[offset + 3 : offset + 6])
        right.extend(colors[offset + 6 : offset + 9])
        back.extend(colors[offset + 9 : offset + 12])
        offset += 12
    down = colors[offset : offset + 9]

    cube = "".join(up + right + front + down + left + back)
    solved = "".join(cube[index + 4] * 9 for index in range(0, 54, 9))
    return cube, solved


def encode_moves(solution: str) -> str:
    result = []
    for move in solution.split():
        if move.endswith("'"):
            result.append(move[0].lower())
        elif move.endswith("2"):
            result.append(move[0] * 2)
        else:
            result.append(move[0])
    return "".join(result)


io = remote(args.HOST or "<HOST>", int(args.PORT or 1337))
io.sendlineafter(b"your choice:", b"3")

for _ in range(50):
    screen = io.recvuntil(b"Please make your choice")
    cube, solved = parse_cube(screen.decode(errors="ignore"))
    steps = encode_moves(kociemba.solve(cube, solved))

    io.sendlineafter(b"your choice:", b"2")
    io.sendlineafter(b"your step:", steps.encode())
    io.sendlineafter(b"your choice:", b"3")

print(io.recvall().decode(errors="ignore"))
```

运行时通过 pwntools 参数传入当前实例：

```bash
python3 solve.py HOST="<HOST>" PORT="<PORT>"
```

每轮检查成功后分数加一；第 50 轮通过时，服务端读取仓库中的 `flag.txt` 并输出：

```text
0xGame{You_are_the_Magic_Cube_Master}
```

第一档只返回一个 Base64 编码的视频地址，第二档只给出 `md5(flag) == 6dba5f8b93e25ca48f897a75bcdf1588`，都不是取得明文 flag 的终点。

## 方法总结

本题的难点不在手工魔方公式，而在表示转换和限时自动化：先从 ANSI 颜色恢复 54 个面片，再把服务端展开图重排为 URFDLB，最后把 Kociemba 记号翻译成服务端动作字符。将解析、求解和协议交互拆开校验，可以避免面序或逆时针符号错误导致整轮失败。
