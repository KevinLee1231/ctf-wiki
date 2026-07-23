# UMDCTF2022 RSI 1 Writeup

## 题目简述

附件 `tutorial.osr` 是 osu! 回放文件。OSR 头部之后包含一段长度字段和 LZMA 压缩的 replay data。本题没有把 flag 编成正常鼠标轨迹，而是把大量空字节与明文 flag 直接放进压缩数据区。

决定性步骤是从游戏回放容器中定位并解压隐藏载荷，归入 `stego`。

## 解题过程

按 OSR 格式依次跳过游戏模式、版本、三个 osu! 字符串、成绩统计、mods、生命条和时间戳。字符串以 `0x00` 表示空值，或以 `0x0b` 加 ULEB128 长度和 UTF-8 数据表示。

时间戳后四字节小端整数是压缩区长度。本地附件解析结果为：

```text
mode: 0
version: 20220222
player: striker4150
compressed length: 53
decompressed length: 278
```

读取 53 字节并用 LZMA 解压：

```python
packed_len = int.from_bytes(fp.read(4), "little")
packed = fp.read(packed_len)
replay_data = lzma.decompress(packed)
print(replay_data.strip(b"\x00").decode())
```

解压结果的大部分是 `0x00`，去除两端空字节后直接得到：

```text
UMDCTF{wE1c0m3_t0_o5u!}
```

## 方法总结

专有容器题应先按格式边界提取目标字段，不能对整个文件盲目解压。OSR 字符串使用 ULEB128 变长长度，若误当成单字节，后续偏移会在较长字段上失效。本题的 replay data 不含正常的逗号分隔帧，反而是一串空字节和明文，这也是它被人为用作隐藏载体的证据。
