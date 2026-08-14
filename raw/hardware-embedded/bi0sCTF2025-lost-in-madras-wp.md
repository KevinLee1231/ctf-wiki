# lost-in-madras

## 题目简述

这是硬件嵌入式赛道的车载总线题，核心是从 CAN 日志恢复两点：

1. 车架号（VIN）；
2. 车辆最终结束点（Landmark Name），最终按 `bi0s{VIN_Landmark}` 输出。

仓库内有：
- `Handout/canlog.txt`（与 found-in-madras 同套日志）；
- `admin/locdeco.py`（经纬度解码脚本）；
- `README.md` 的任务说明与短 writeup。

`locdeco.py` 给出了 GPS 位解码规则（经纬度位段与偏置、换算公式），是唯一可直接复用的解析代码证据。

## 解题过程

### 关键观察

- README 与短 writeup 明确 VIN 使用 UDS 请求/应答报文，位置/坐标使用 DBC 换算。
- `locdeco.py` 中的 `decode_gps()` 提供固定公式：纬度按 8/6/14 位、经度按 9/6/14 位拆分，再做 `-89`、`-179` 偏置。
- 仓库中未附 DBC，无法只靠附件重新证明“坐标信号为何对应 CAN ID `0x465`”；不过官方 `locdeco.py` 已固定给出该帧的 8 字节数据及位段公式，因此数值解码仍可在本地闭环复算。

### VIN 复现说明

在日志片段中可见 VIN 报文片段（`vcan0 73B`）：

```text
vcan0  733   [8]  03 22 F1 90 00 00 00 00
vcan0  73B   [8]  10 14 62 F1 90 31 46 4D
vcan0  733   [8]  30 00 00 00 00 00 00 00
vcan0  73B   [8]  21 48 4B 37 44 38 32 42
vcan0  73B   [8]  22 47 41 33 34 39 35 34
```

按 UTF-8 回拼后可得到 VIN：

```text
VIN = 1FMHK7D82BGA34954
```

### 坐标与 flag 复核

日志中与脚本输入完全相同的帧为：

```text
vcan2  465   [8]  66 0D F4 48 1A 0E DD 00
```

`locdeco.py` 的核心公式（已内嵌）：

$$lat = (\text{deg}_{lat} - 89) + \frac{min_{lat}+minDec \times 0.0001}{60}$$
$$lon = (\text{deg}_{lon} - 179) + \frac{min_{lon}+minDec \times 0.0001}{60}$$

使用仓库固定的 Windows 虚拟环境执行官方 `locdeco.py`，输出为：

```text
Latitude: 13.06334, Longitude: 80.27935
```

该坐标对应 M. A. Chidambaram Stadium，并与仓库给出的最终 flag 相互核验：

```text
flag: bi0s{1FMHK7D82BGA34954_M. A. Chidambaram Stadium}
```

DBC 缺失只影响“如何从车型映射到 `0x465`”这一前置识别步骤，不影响上述官方帧、位段计算和最终坐标的复算。

## 方法总结

- 核心技巧：先按 ISO-TP 流控重组 VIN，再用 DBC/官方解码器定位并解析坐标帧；两个阶段应分别保留可核验证据。
- 识别信号：出现 `F1 90` 这类 VIN 请求/响应模式时，应优先先恢复 ASCII VIN 再做地理反解。
- 复用要点：无 DBC 时可分层记录：第一层（报文解析与 VIN），第二层（模型化坐标换算）。不要把缺失文件当作可猜测点强行补全。
