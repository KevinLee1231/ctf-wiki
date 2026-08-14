# found-in-madras

## 题目简述

这是硬件嵌入式赛道的车载总线题，核心是从 CAN 日志恢复两类结果：

1. 车辆最高速度（`bi0s{max speed_SEED_KEY}` 结构）；
2. 安全访问场景中的 `SEED` 与 `KEY`。

仓库内可见：
- `Handout/canlog.txt`（约 38MB）；
- `admin/get_speed.py`（速度提取脚本）；
- `README.md` 的任务流程与短 writeup；
- 官方 flag 与题面提示。

`admin/get_speed.py` 的逻辑是“用 CAN ID `423` 中的数据拼接速度字节，再按量程/偏置换算”。该脚本默认打开 `canlog6`，与实际文件名 `canlog.txt` 不一致，是一处已记录的实现偏差。

## 解题过程

### 关键观察

- 速度帧可见 ID 为 `423`，日志中该 ID 行格式接近 `[5] xx yy 00 00 00`。
- 题库短 writeup 明确了 `speed = max(...)` 及安全访问 seed/key 的思路。
- `canlog.txt` 中存在明显的 UDS 安全访问会话相关帧，其中可见 seed 与 key 的候选字节串。

### 复现实例化说明

#### 1) 速度恢复（官方脚本修正）

`get_speed.py` 原脚本片段：

```python
with open('canlog6', 'r') as file:
    ...
    if "vcan1" in line and "423" in line:
        parts = line.strip().split()
        low = parts[3]
        high = parts[4]
        speed = high + low
```

实际文件为 `canlog.txt`，若直接运行应先对齐输入文件名或复制重命名。

修正后的核心逻辑仍是：

$$v = \left(\texttt{int}(high+low,16)\right)\times 0.01 - 100$$

按此规则对全部 `vcan1`、`423` 帧取最大值，得到：

```text
max speed = 65.13
```

#### 2) SEED / KEY 提取

在 `canlog.txt` 中能找到多组安全访问尝试。决定性的一组报文是：

```text
vcan0  733   [8]  02 27 15 00 00 00 00 00  # 请求 level 0x15 的 seed
vcan0  73B   [8]  06 67 15 35 77 94 86 00  # 正响应，seed = 35779486
vcan0  733   [8]  06 27 16 CA 43 AB BE 00  # 提交 key = CA43ABBE
vcan0  73B   [8]  02 67 16 00 00 00 00 00  # 正响应，key 获准
```

UDS 的正响应 SID 是请求 SID `0x27` 加 `0x40`，即 `0x67`；最后的 `67 16` 证明这一组 key 确实通过，而不是从大量失败尝试中随意挑出的候选。因此 `SEED=35779486`、`KEY=CA43ABBE`。

### 最终核验

```text
flag: bi0s{65.13kph_35779486_CA43ABBE}
```

## 方法总结

- 核心技巧：先修正脚本输入路径，再按固定字段规则（CAN ID + 字节拼接）恢复数值；其后再做安全访问相关 UDS 帧映射。
- 识别信号：出现 `vcan1` + 特定 ID + 长度字段时，先统一解析格式再谈高级协议语义，否则易错读位序。
- 复用要点：保留原始报文与解析公式，能快速复核字节序和 UDS 状态；脚本硬编码文件名与实际附件不一致时，应明确修正输入映射，而不能假称原脚本直接运行成功。
