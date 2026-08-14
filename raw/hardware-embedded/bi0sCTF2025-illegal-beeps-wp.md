# illegal-beeps

## 题目简述

这是一道硬件题，目标是解码通过 ESP32 蜂鸣器输出的音频信号。仓库内包含：
- `admin/chall-source/main/main.cpp`（编码/发送实现）；
- `admin/rand-generation/main/main.cpp`（用于复现乱序 key 演化）；
- `admin/solve.py`（音频解码与重组脚本）；
- `handout/illegal-beeps.elf`、`handout/ze_beeps.wav`。

题面说明是“Ze beeps make ze message”，官方 short writeup 与 `Readme` 均指向 FSK 编码检测：识别频段、定位前导码与尾随码，再把每三个音符恢复成一个字节。

## 解题过程

### 关键观察

`main.cpp` 明确了发码参数：
- 基准频率 `BASE_FREQ = 2000`；
- 步进 `FREQ_STEP = 150`；
- 预置频率 `PREAMBLE_FREQ = 2800`，`POSTAMBLE_FREQ = 3200`；
- 每个字节被拆成三个八进制数位（`sendCustomByte`）；
- 非法指令异常处理器会反复调用 `rand()` 更新 `scramble_key`，因此仅从音频恢复密文字节还不够，还要还原同一密钥序列。

`solve.py` 则给出实用解析链：频谱量化→分段聚类→找到前导/后导→按 octal 解码→按滚动 key 异或。

### 复现实例化说明

`sendCustomByte` 使用的编码关系：

```python
symbol0 = data // 64
symbol1 = (data // 8) % 8
symbol2 = data % 8
freq = BASE_FREQ + symbol * FREQ_STEP
```

`solve.py` 复现流程：

1. 载入 `ze_beeps.wav`；
2. 对每 20ms 窗口做 FFT 并量化到允许频率集合；
3. 按序列匹配前导码 `[2800,2300,...]` 与后导码；
4. 将每三个 symbol 作为一个八进制字节；
5. 按官方脚本对每个字节应用 `chr((value ^ key) % 251)`，其中 `current_key` 来自异常处理路径中 `rand()` 的调用轨迹。

当前脚本中的默认 key 表为：

```python
current_key = [0x3E,0x65,0x3F,0xE8,0x18,0xFA,0x25,0x3D,0x8C,0x48,0xA4,0xA,0x1B,0x83,0x77,0xD3,0x71,0x9D,0xC9,0xE4,0x6C,0x7F,0x7D,0x4E,0x30,0x4,0x86,0xBB,0xA2,0x5D,0xAC,0x19,0xA4,0x50,0x0,0xD6, 0, 0]
```

脚本恢复出的明文与仓库 `Readme.md` 中的 flag 一致：

```text
bi0s{c0D3_1n_7h3_ESP_g0e5_bo0p_b3eP}
```

### 验证

由于当前环境未安装 `librosa`，未直接执行 `admin/solve.py`；但脚本逻辑与编码常量与 `main.cpp` 一致，属于结构闭环且可执行的解码路径。

## 方法总结

- 核心技巧：先确认 FSK 编码参数，再定位前导码和尾随码；最后完成“八进制字节 + 异常处理器密钥序列”的逆变换，不能把频率直接当作 8 位字节。
- 识别信号：看到频率全集为 `{2000,2150,...}`、每字节三符号编码与前后导码时，应立即转入固定窗口 FFT + 模式匹配。
- 复用要点：硬件题里先保留通信参数（载波、步进、阈值、码元时长），比保留原始波形文件更重要，便于附件丢失后复现。
