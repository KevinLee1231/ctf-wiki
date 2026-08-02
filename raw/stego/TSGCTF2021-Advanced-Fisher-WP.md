# TSGCTF2021 Advanced Fisher WP

## 题目简述

题目给出一个约 21.5 秒的 `result.wav`。原始数据先把 flag 编成摩尔斯码的 0/1 信号，每一位扩展为 2000 个采样点：0 写成静音，1 写成采样率 44100 Hz 下的 440 Hz 正弦波。随后生成器对全部采样点执行 `numpy.random.shuffle`，彻底打乱时间顺序。

直接播放或查看时域波形都无法读出摩尔斯码，但打乱只改变位置，不改变各采样值的出现次数。440 Hz 正弦波在 44100 Hz 采样率下以 2205 个采样为一个离散周期；统计每个相位值的频数，仍能恢复原始 0/1 序列相邻位置之间的差分。这是利用音频采样直方图重组隐藏信息的隐写题。

## 解题过程

生成器的核心是：

```python
signals = string_to_signals(flag)
wave = np.empty(len(signals) * 2000)

for i in range(len(wave)):
    if signals[i // 2000] == 0:
        wave[i] = 0.0
    else:
        wave[i] = np.sin(i * 440 / 44100 * (2 * np.pi))

np.random.shuffle(wave)
```

`string_to_signals` 用 1 个采样块表示点、3 个连续的 1 块表示划，码内间隔是 1 个 0 块，字符间隔最终形成 3 个 0 块。flag 一共编码成 473 位信号。

因为

$$
\gcd(440,44100)=20,\qquad \frac{44100}{20}=2205,
$$

正弦采样值每 2205 点重复。枚举一个周期的理论值，再将 WAV 中每个非零采样按误差阈值映射到相位，便可得到长度 2205 的频数数组：

```python
mapping = {}
for i in range(2205):
    value = np.sin(i * 440 / 44100 * (2 * np.pi))
    mapping[value] = i

counts = [0] * 2205
cache = {}
for frame in frames:
    if frame == 0:
        continue

    if frame in cache:
        phase = cache[frame]
    else:
        phase = None
        for value, candidate in mapping.items():
            if np.abs(frame - value) < 1e-6:
                phase = candidate
                cache[frame] = phase
                break
        if phase is None:
            raise ValueError("unmatched sample")

    counts[phase] += 1
```

每个逻辑位占 2000 点，而 $2000=5\times400$、$2205=5\times441$。因此取相位下标 $1,6,11,\ldots$，把问题缩成长度 441 的循环数组；对相邻频数做差，再按乘 400 模 441 的置换重排，就能得到原始信号的跃迁量：

```python
shrink_counts = [counts[i] for i in range(1, 2205, 5)]
count_diffs = [
    shrink_counts[i] - shrink_counts[i - 1]
    for i in range(len(shrink_counts))
]
reordered_diffs = [
    count_diffs[i * 400 % len(count_diffs)]
    for i in range(len(count_diffs))
]
```

频数只给出循环差分，开头还存在一个常量偏移。flag 格式保证前缀为 `TSGCTF{`，先把该前缀也编码成信号，用它扣除已知部分造成的跃迁；随后从已知状态 1 开始对差分积分，即可恢复剩余 0/1 序列：

```python
leading = [0] + string_to_signals("TSGCTF{") + [0]
for i in range(len(leading) - 1):
    reordered_diffs[i] -= leading[i + 1] - leading[i]

value = 1
signals = []
for i in range(len(leading) + 2, 474):
    diff = reordered_diffs[i % len(reordered_diffs)]
    signals.append(value)
    value += diff

print("TSGCTF{" + signals_to_string(signals))
```

最终解码得到：

```text
TSGCTF{THE-TRUE-F1SHERM4N-U53S-M0RSE-CODE}
```

## 方法总结

随机打乱序列并不等于销毁全部结构：只要样本值与其原始相位存在确定关系，直方图仍会泄漏信息。本题先利用 $440/44100$ 的最大公约数确定 2205 点周期，再利用 2000 点逻辑块与周期的模关系，把相位频数差转换成相邻摩尔斯位的变化。已知 flag 前缀用于消除循环差分的初始不确定性。分析这类“洗牌后音频”时，应分别检查时序、频谱、样本值分布和周期相位；这里真正保留下来的信道是采样值频数，而不是可听内容。
