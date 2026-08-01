# PRISM's end BLACKBOX

## 题目简述

这是一个 TLS 上的 raw TCP 黑盒交互题。连接后先出现 `mirror token`，必须把 token 整体逆序回传，之后才能使用命令行接口。

握手后可以使用的核心命令如下：

- `look ids <face>`：查看某一面的 25 个贴纸 ID；
- `scan ids`：查看当前 24 个扫描槽中的有序 ID；
- `oracle`：查看当前封印的目标、步数预算和剩余查询次数；
- `shadow <turn>`：不实际执行转动，只返回该命令执行后状态的摘要；
- `turn <turn>`：实际执行一次转动；
- `verify`：验证当前 opening。

整个物体可以建模成一个 5×5×5 立方体表面的 150 张独立贴纸。可执行动作只有：

- 三个物理轴 `x / y / z`；
- 每个轴的三层 `-2 / 0 / +2`；
- 每层两个四分之一转方向 `-1 / +1`。

因此物理动作一共 `3 × 3 × 2 = 18` 个。服务端另外施加了一条重要限制：相邻两步不能绕同一个物理轴旋转，即使转的是不同层也不行。

每个 opening 都会重新随机化文本命令 `roll / pitch / yaw`、`low / mid / high` 和正负方向到上述 18 个物理动作的映射。七道封印各有三个 opening，总计需要连续完成 21 组随机实例：

| 封印 | `goal_type` | 目标规模 | 步数预算 |
|---|---|---:|---:|
| 1 | `selected_slots` | 10 个指定槽精确匹配 | 12 |
| 2 | `required_ids` | 10 个 ID 全部进入扫描区 | 15 |
| 3 | `selected_slots` | 21 个指定槽精确匹配 | 17 |
| 4 | `locked_match` | 锁定槽与目标槽同时精确匹配 | 18 |
| 5 | `cycle_program` | 按循环程序变换，固定槽不变 | 20 |
| 6 | `local_delta` | 按循环程序变换，锚点不变 | 22 |
| 7 | `full_scan_match` | 24 个扫描槽全部精确匹配 | 19 |

## 解题过程

### 1. 完成镜像握手并恢复完整状态

客户端先读取到提示符 `> `，用正则提取 `mirror token`，逆序后发送。通过握手后，依次执行：

```text
look ids front
look ids back
look ids left
look ids right
look ids top
look ids bottom
```

按 `front, back, left, right, top, bottom` 的顺序拼接结果，得到长度为 150 的状态数组。程序会额外验证它恰好是 `0..149` 的一个排列，避免把提示文本中的其他数字误当成贴纸 ID。

不能把扫描槽写死成第一次观察到的位置。正确做法是同时读取完整状态和 `scan ids`，建立：

```python
id_to_position = {
    sticker_id: position
    for position, sticker_id in enumerate(state)
}
scan_positions = tuple(
    id_to_position[sticker_id]
    for sticker_id in scan_ids
)
```

这样每个 opening 都能重新恢复“扫描槽序号 → 立方体物理位置”的有序映射。

### 2. 建立 150 贴纸的物理模型

每个表面位置表示为：

```text
(x, y, z, nx, ny, nz)
```

前三项是坐标，后三项是该贴纸所在面的法向量。坐标范围是 `-2..2`。例如前表面满足 `z = 2`、法向量为 `(0, 0, 1)`。

一次物理动作只旋转满足指定层坐标的贴纸。例如绕 `x` 轴正向旋转时：

```text
(x, y, z) -> (x, -z, y)
```

法向量应用同样的旋转。把新坐标和新法向量映射回六个面的行列位置，就能预计算每个动作对 `0..149` 的置换。之后无论跟踪完整状态还是少量目标贴纸，都只需查置换表。

### 3. 用五次 `shadow` 查询恢复随机 wiring

`shadow` 返回 24 个十六进制字符。实验发现它是完整状态的 BLAKE2s-96：

```python
hashlib.blake2s(
    bytes(full_state),
    digest_size=12,
    person=b"PRISMv2",
).hexdigest()
```

预先枚举 18 个物理动作后，可以分别模拟动作并计算摘要，再与服务端 `shadow` 返回值比较，从而识别一条文本命令对应的物理动作。

每个 opening 只有五次 `shadow` 额度，恰好可以用以下命令恢复整套映射：

```text
roll low +
roll mid +
roll high +
pitch low +
yaw low +
```

前三次确定 `roll` 对应的物理轴、方向，以及 `low / mid / high` 到 `-2 / 0 / +2` 的层置换；后两次确定 `pitch`、`yaw` 对应的另外两个轴及方向。负方向由正方向取逆即可。最终得到完整的：

```text
文本命令 <-> 18 个物理动作
```

双射。

这里必须把“文本 wiring”和“物理状态搜索”彻底分开。搜索器始终输出物理动作，只有真正发送到服务器前才把物理动作翻译成当次 opening 的文本命令。

### 4. 把七类 oracle 统一成位置约束

对于精确目标，先根据目标 ID 找到该贴纸当前的物理位置，再把目标槽转换为 `scan_positions[slot]`。于是问题统一为：

```text
start = 目标贴纸当前所在位置
goal  = 这些贴纸最终必须到达的位置
```

各类型的处理如下：

- `selected_slots`：直接使用 `selected_slots` 和 `target_ids`；
- `locked_match`：把 `locked_slots/locked_ids` 与 `target_slots/target_ids` 合并；
- `cycle_program`：从 `base_ids` 开始，按 `cycle_order = left_to_right` 从左到右应用循环，再约束所有循环槽和 `fixed_slots`；
- `local_delta`：与上一类相同，但固定项字段改为 `anchors`；
- `full_scan_match`：约束全部 24 个扫描槽；
- `required_ids`：不要求固定顺序，只要求所有 `required` ID 最终都位于扫描位置集合中。

循环 `(a b c)` 的方向按“源位置的值移动到下一个位置”解释：

```text
a -> b
b -> c
c -> a
```

第 5、6 道的 `verify` 结果可以确认，该循环方向与从左到右的组合顺序正确。

### 5. 搜索策略

前期精确目标规模较小时使用 IDA*。启发函数来自反向 BFS 构造的模式数据库：

- 所有二贴纸模式；
- 若干高价值三贴纸模式；
- 后期再加入四贴纸模式；
- 状态数允许时加入最多两个五贴纸模式。

每张模式库给出该子模式到目标的精确距离，取所有模式距离的最大值作为可采纳下界。扩展节点时禁止与上一步同轴，并优先搜索启发值较小的后继。

第 2 道 `required_ids` 是集合目标，直接跟踪贴纸身份会引入大量无意义排列。脚本改用 150 个位置占用布尔量进行 SMT：

```text
occupied[step][position]
```

每一步按选定物理置换更新占用集合，最终约束扫描位置集合覆盖全部必需贴纸。这样求解器不需要关心这些贴纸在扫描区内的相对顺序。

第 4 至第 7 道只要求在预算内找到任意解，不要求最短。继续用 IDA*证明较浅深度无解会出现指数爆炸，因此改用有界正反向搜索：

1. 从目标反向完整展开 5 层，保存约 10 万到 35 万个目标边缘状态；
2. 从起点按层进行 beam search，每层最多保留 50000 个状态；
3. 排序主键依次为模式库距离总和、错位贴纸数、可采纳下界；
4. 用 `bytes(state) + previous_axis` 作为紧凑去重指纹；
5. 前向状态命中反向边缘时，检查拼接处两步是否同轴；
6. 将反向路径逆序并逐步取逆，拼成完整物理路径；
7. 严格限制最终长度不超过 oracle 的 `turns_left`。

这一步是稳定求解后期封印的关键。真实运行中，第 6 道曾找到一条 21 步路径并在 22 步预算内通过；最终第 7 道三组分别在 11、9、10 步左右完成。

### 6. 执行前后的双重校验

搜索得到物理路径后，程序逐步：

1. 用当次 wiring 翻译为文本命令；
2. 发送 `turn`；
3. 在客户端维护的完整状态上应用同一物理置换。

全部动作执行完后，再次读取 `scan ids`，首先检查它与客户端预测值完全一致，然后检查当前 oracle 的槽位条件，最后才发送 `verify`。这可以在消耗 opening 前及时发现 wiring、坐标方向或循环方向错误。

运行命令如下。目标地址会变化，因此通过 `--host` 传入当前实例域名：

```bash
python -u solve.py \
  --host "<challenge-host>" \
  solve \
  --max-openings 21 \
  --max-depth 22 \
  --engine auto
```

最终服务端输出：

```text
seal 7/7 opens. the last lens clears.
the prism opens.
d3ctf{8d761350-a9f9-1244-d6bf-13950377c1f4}}
```

服务端输出末尾多打印了一个 `}`。按标准 flag 格式取到第一个右花括号，最终 flag 为：

```text
d3ctf{8d761350-a9f9-1244-d6bf-13950377c1f4}
```

## 方法总结

本题表面上是黑盒命令交互，实质是“随机命令编码下的受限置换搜索”。完整解法可以概括为四层：

1. **协议层**：逆序 mirror token，稳定解析六面状态、扫描槽和 oracle；
2. **识别层**：识别 BLAKE2s-96 的 shadow 摘要，用五次查询恢复每个 opening 的随机 wiring；
3. **模型层**：以坐标和法向量建立 150 贴纸、18 个物理动作的精确置换模型；
4. **搜索层**：小目标使用 PDB + IDA*，集合目标使用 mask-SMT，后期预算目标使用 5 层反向边缘 + 50000 宽度 beam search。

最重要的工程判断是：题目后期只要求“预算内可行”，不要求“最短”。用最优搜索证明浅层无解会浪费大量时间，而有界正反向搜索能利用服务器给出的较宽预算，在可控内存内稳定找到路径。

另一个容易踩坑的点是随机 wiring。若直接在 `roll/pitch/yaw` 文本空间搜索，每个 opening 都要重建模型；把物理动作与文本命令分层后，立方体置换、模式库和搜索算法都可以复用，只需在执行前做一次翻译。
