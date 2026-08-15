# Strom

## 题目简述

题目提供一份浏览器观看视频时产生的网络流量。HTTP 请求中出现一个 HLS 播放列表和连续的 MPEG-TS 分片，flag 只在视频中间的少数帧出现，需要先重建完整视频。

## 解题过程

检查 HTTP 请求可以看到一个 `.m3u8` 文件，以及编号从 0 到 14 的 15 个 `.ts` 分片：

```text
/WyK2SW5mcYDArna2IlwZ4C4SwDjZ717a.m3u8
/WyK2SW5mcYDArna2IlwZ4C4SwDjZ717a0.ts
...
/WyK2SW5mcYDArna2IlwZ4C4SwDjZ717a14.ts
```

可以用 Wireshark 的“Export Objects / HTTP”导出，也可以直接使用 Tshark：

```bash
mkdir -p http-objects
tshark -r strom.pcapng --export-objects http,http-objects
```

播放列表和分片位于同一目录时，FFmpeg 会按照 `.m3u8` 中的顺序读取所有片段：

```bash
ffmpeg \
  -i http-objects/WyK2SW5mcYDArna2IlwZ4C4SwDjZ717a.m3u8 \
  -c copy recovered.mp4
```

逐帧检查视频中段，或者先导出中间帧，可以看到一帧中的文字：

```text
shellmates{57R34m1nG_w1Th_hL5}
```

该帧只有可直接转写的文字，不包含影响解法的额外视觉结构，因此不把视频截图作为 WP 资源保留。

## 方法总结

HLS 取证不能只恢复某一个 `.ts` 文件：`.m3u8` 决定分片顺序和时长，完整复原应同时导出播放列表及全部分片，再交给支持 HLS 的工具拼接。最终 flag 为 `shellmates{57R34m1nG_w1Th_hL5}`。
