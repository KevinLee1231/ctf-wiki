# Ping Pong

## 题目简述

题目给出一份网络抓包 `ping-pong.pcapng`，提示“ping is talking”。关键不是 ICMP 头字段，而是按顺序拼接 echo request 中的原始 `data` 字段，还原被拆散的文件。

## 解题过程

先只筛选 ICMP echo request（`icmp.type == 8`），避免把响应中的重复载荷再次拼入结果。`tshark` 输出十六进制数据，`xxd` 负责还原字节流：

```bash
tshark -r ping-pong.pcapng \
  -Y 'icmp.type == 8' \
  -T fields -e data \
  | tr -d '\n' \
  | xxd -r -p > recovered.png

file recovered.png
```

本地复核中共命中 5097 个 echo request，拼接结果是一个 `975 × 163`、RGBA、非隔行扫描的 PNG。图中只有一行放大的文本：

```text
grey{5n34ky_p1ng5}
```

该画面没有超出文字本身的独立视觉证据，因此在归档 WP 中直接转写内容，不保留纯 flag 截图。

## 方法总结

处理 ICMP 隐蔽载荷时应先确认请求与响应是否携带相同数据，再决定过滤方向；否则文件往往会因重复拼接而损坏。本题的稳定流程是“按抓包顺序过滤 echo request → 提取 `data` → 十六进制还原 → 用文件头验证结果”，最终 flag 为 `grey{5n34ky_p1ng5}`。
