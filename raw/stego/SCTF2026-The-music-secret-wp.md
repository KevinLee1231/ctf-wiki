# The music secret

## 题目简述

附件只有 `stream.pcapng`。流量中包含 studio 与 edge-cache 两路同源 WAV 的 HTTP Range 请求，以及一组描述节拍、密钥调度、同步字、纠错和帧校验的元数据。flag 并不直接存在于某个 HTTP 对象中，而是编码在两路音频的采样差分里。

完整链路为：按 `Content-Range` 重组两路 WAV → 按 cache Range 首次出现顺序派生 SHA-256 密钥 → 在固定节拍网格上量化音频差分 → 识别同步字和帧头 → Hamming(7,4) 纠错并用 CRC16 验证 → 按 index 重排 → XOR 和 Base64 解码。

## 解题过程

### 1. 从 HTTP Range 重组两路 WAV

会话接口给出 `track_id=7`、`bpm=120`、`subdivision=2`，响应头还明确写出：

```text
X-Key-Schedule: sha256(agent|id|csv(first-seen-range-start))
X-Stream-FEC: hamming-7-4
X-Frame-Sync: 8-symbol
X-Frame-Fields: index,length
X-Frame-Check: crc16
X-Frame-Order: indexed-blocks
```

抓包处于 late-start、quota-stop 状态，单个响应可能截断。对 `/edge/v1/assets/7/studio` 和 `/edge/v1/assets/7/cache` 的每个 206 响应，解析：

```text
Content-Range: bytes start-end/total
```

将实际收到的 body 写入目标缓冲区的 `[start,end]`，后续重叠 Range 用来补洞，不能按抓包到达顺序简单拼接。补齐后得到两份参数一致的单声道 IEEE float32 WAV，采样率均为 8000 Hz，这保证逐采样差分有意义。

### 2. 从请求顺序派生密钥

只扫描 cache 资源的请求，按 PCAP 中首次出现的时间顺序提取 `Range: bytes=start-end` 的 `start`。重复 start 只保留第一次，绝对不能排序。附件中共有 280 个唯一 start，前几项为：

```text
3194880,2686976,3817472,3932160,1948160,
2244608,1277952,3096576,1359872,1949696
```

把完整序列拼成：

```text
StreamClient/2.1|7|3194880,2686976,3817472,...
```

计算 SHA-256。正确密钥的前 8 字节应为：

```text
a679df158ff5d930
```

这一步是独立检查点；若前缀不符，通常是误把 studio 请求算入、保留了重复 start，或对 start 做了数值排序。

### 3. 提取四元差分符号

120 BPM 表示每秒 2 拍，每拍再分成 2 个子单位，所以每秒 4 个符号。8000 Hz 采样率下，每个符号长度为

$$
\frac{8000}{(120/60)\times2}=2000
$$

个采样点。

对齐两路 WAV，计算 `cache - studio`。有效变化落在 2000-sample 网格上，每个块中心的稳定窗口可聚成四个幅值等级，官方分析中归一化中心值约为 `9, 19, 29, 39`。按从低到高映射为四元符号 $0,1,2,3$，得到主符号流。

### 4. 同步、纠错与解密

在四元流中搜索重复的 8 符号同步字：

```text
(3, 1, 0, 3, 2, 1, 0, 1)
```

同步字后按题面约束解析 `index` 和 `length`，再把后续四元符号展开为比特。每 7 位做 Hamming(7,4) 解码，纠正一位错误并还原有效 nibble；随后按帧内 CRC16 检查结果。CRC 连续通过才说明幅值映射、位序和帧边界都正确，不能用“输出看起来像文本”代替校验。

按 `index` 重排有效载荷并连接，使用第二步 SHA-256 摘要循环 XOR。结果是一段 Base64 文本，再解码得到：

```text
SCTF{stream_order_meets_noisy_rhythm}
```

## 方法总结

这道题虽然以 PCAP 开始，但普通流量重组只是证据恢复层，决定性秘密位于两路音频的差分隐蔽信道，因此归入 stego。解题时应充分利用协议元数据给出的约束：Range 顺序确定密钥，BPM 确定符号周期，同步字确定帧起点，Hamming 负责纠错，CRC16 负责否定错误假设。任何一层都不应靠听感或肉眼猜测，阶段性前缀与 CRC 才是可复现的判断依据。
