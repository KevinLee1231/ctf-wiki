# Miku Music Machine

## 题目简述

程序要求输入 50 字节。每个字节先与固定表异或，再从低位到高位拆成四个 2-bit 方向：

```text
0 = 上，1 = 右，2 = 下，3 = 左
```

因此完整输入编码 200 步迷宫路径。题目的特殊之处是没有显式的墙判断：当前位置对应一个函数指针，程序执行 `cells[cur]()`，利用 Windows eXtended Flow Guard（XFG）决定该函数能否作为间接调用目标。

## 解题过程

### 1. 把 XFG 表解释成迷宫墙

XFG 会在间接调用前检查目标是否登记为合法函数，以及目标函数的类型哈希是否与调用点一致。题目后处理程序从 XFG/CFG 表中移除了墙格对应的 `cell<N>`。

因此：

- 表中存在且类型哈希正确的 cell 是可走格；
- 不在表中的 cell 是墙；
- 走进墙时，XFG 快速失败，进程以 `STATUS_STACK_BUFFER_OVERRUN` 终止。

可以从 `cells` 数组恢复每个格子的函数地址，再解析 PE load config 中的 XFG/CFG 目标表，得到静态迷宫骨架。

### 2. 识别开关与门

普通 `cell<N>` 含固定的 7 个 NOP。开关函数则包含：

```asm
xor byte ptr [rip + offset], imm8
```

它会修改另一格函数前的 XFG 类型哈希低字节。被修改的格子就是门：开关触发后，该门从非法间接调用目标变为合法，或反向关闭。

仓库官方恢复出的 21×21 迷宫用小写 `a..h` 标记开关，用大写字母标记对应门。有效路径按字母顺序访问所有开关，最后到达右下角。

### 3. 求 200 步路径并打包

每触发一个开关，就把对应大写门改成空地，再搜索到下一个开关。官方 patcher 使用固定的方向顺序执行 DFS；迷宫被设计为每个阶段只有一条有效最短路径。

每四步打包为一个字节：

```python
packed = (
    move[0]
    | move[1] << 2
    | move[2] << 4
    | move[3] << 6
)
```

程序执行的是：

```text
packed = input_byte XOR XOR_TABLE[i]
```

所以最终输入为：

```python
input_byte = packed ^ XOR_TABLE[i]
```

### 4. 恢复输入

按官方 `maze.txt`、patcher 的路径算法和 `main.c` 中 50 字节 XOR 表复算，得到：

```text
SEKAI{https://www.youtube.com/watch?v=J---aiyznGQ}
```

这里的 URL 形字符串就是 flag 本身，必须原样保留；它不是解题依赖的外部资料。

题目另附 `mmm-v2.exe`，用于替代在较新 Windows 版本上行为变化的原版 XFG 检查。两版使用相同迷宫和 flag。

## 方法总结

本题把控制流完整性元数据当作关卡数据：

```text
合法 XFG 目标 = 可走格
非法 XFG 目标 = 墙
运行时改写类型哈希 = 开关门
```

逆向时不能只看函数主体；PE load config、guard function table 和函数前的类型哈希同样属于程序语义。恢复路径后还要严格按“每字节四步、低位优先、小端 2-bit”打包，再与表异或。
