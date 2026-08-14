# water tank

## 题目简述

这是一道 Modbus TCP 工控题。附件 `modbus_capture.pcap` 中泄露了控制系统凭据，远端服务则在前 50 个线圈中随机选择 3 个阀门，并随机规定每个阀门由 `0` 还是 `1` 表示关闭。只有把三个阀门全部置于关闭态，服务端才会把 flag 写进 holding registers。

仓库同时提供 `admin/src/server.py` 与 `admin/solution/solver.py`。服务端源码足以确定状态机；官方 solver 展示了差分探测思路，但它与当前服务端版本存在两处不一致，不能把“脚本存在”写成“已直接验证通过”。

## 解题过程

先检查抓包中的应用数据。第 6172 个数据包的原始负载包含：

```text
operator:kjYTV^%Gh5csd
```

这与 `admin/src/.env` 中的 `USERNAME`、`PASSWORD` 一致。当前公开的 `server.py` 只读取 `FLAG`，并没有实现登录层，因此凭据应由比赛部署时的外层接入服务消费；WP 不能把它误写成 Modbus 协议自带认证。

服务端每次启动执行：

```python
VALVE_COIL_ADDRESSES = random.sample(range(0, 50), 3)
VALVE_LOGICS = [random.choice([0, 1]) for _ in VALVE_COIL_ADDRESSES]
```

对第 $i$ 个阀门，关闭条件是：

```python
is_closed = coil_value == 1 if logic_type == 0 else coil_value == 0
```

因此 `logic_type=0` 时写 `1` 才关闭，`logic_type=1` 时写 `0` 才关闭。若尚有 $n$ 个阀门开启，每轮水位增加量为 $\Delta L=10n$；改变一个真实阀门的线圈状态，会使相邻采样间的增量变化 10，而无关线圈不会改变增量。

可据此逐一探测地址 `0..49`：

```python
closed = {}
for addr in range(50):
    # 已识别阀门始终保持关闭，其他未知线圈先置 0。
    delta0 = measure_level_delta(addr, 0, closed)
    delta1 = measure_level_delta(addr, 1, closed)
    if delta0 < delta1:
        closed[addr] = 0
    elif delta1 < delta0:
        closed[addr] = 1
    if len(closed) == 3:
        break
```

`measure_level_delta` 应读取 `CURRENT_TANK_LEVEL_HR = 12`，等待一个完整更新周期后再次读取；得到较小增量的写值就是该阀门的关闭值。识别出的阀门在后续探测中必须一直保持关闭，否则其他阀门造成的变化会污染基线。

三个阀门均关闭后，`server.py` 把 flag 每两个字符打包为一个 16 位大端寄存器，并从随机位置 `50..150` 写入。扫描 holding registers `50..149`，将每个寄存器转为两个大端字节，再搜索 `bi0s\{.*?\}` 即可得到：

```text
bi0s{n0_m0r3_w3t_p4nts_0n_my_w4tch}
```

需要特别记录官方 solver 与当前服务端的版本偏差：solver 只探测 `15..49`，但服务端会从 `0..49` 选阀门；solver 的 `get_tank_status()` 读取寄存器 `0..1`，而当前服务端把水位更新到寄存器 `12`。所以随附脚本表达了正确的差分思路，却需要把探测范围改为 `0..49`、把水位读取地址改为 `12` 后，才与仓库当前服务端一致。本次未安装缺失的 `pymodbus==3.9.2`，也没有启动持续服务，因此这里只声明源码级闭环与抓包凭据核验，不冒充远端运行通过。

## 方法总结

这道题的决定性步骤是对随机线圈做差分探测：通过水位增量是否变化识别真实阀门，再由增量较小的一侧判断正逻辑或反逻辑。PCAP 负责恢复接入凭据，Modbus 状态机负责恢复阀门映射，两条证据链不能混为一谈。

复用到其他 ICS 题时，应先从服务端或抓包确认寄存器地址、更新周期和数值尺度，再写探测器。官方 solver 与服务端版本不一致时，必须沿实际状态变量核对读写地址，不能仅凭脚本文件名宣称利用已验证。
