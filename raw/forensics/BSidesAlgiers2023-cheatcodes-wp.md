# CheatCodes

## 题目简述

题目提供 `cheat.pcapng`，背景是玩家怀疑朋友在 GTA V 中使用了作弊码，要求按启用顺序恢复三个作弊码名称。流量不是键盘记录，而是 USB 游戏手柄的 HID 报告，因此需要从 `usb.capdata` 中还原按键序列。

## 解题过程

长度为 112 字节的数据帧包含手柄的输入报告，可以先用 Tshark 导出：

```bash
tshark -r cheat.pcapng \
  -Y "frame.len == 112" \
  -T fields -e usb.capdata > controller-data.txt
```

报告中偏移 4 到 5 字节是按钮位图，后续两个 16 位字段对应左右扳机。位图的低 16 位依次表示方向键、肩键、摇杆按键、扳机和 `A/B/X/Y`。解析时还要忽略全零的松开帧，并对连续相同状态去重：

```python
KEYS = [
    "up", "down", "left", "right",
    "LB", "RB", "LSP", "RSP",
    "", "", "LT", "RT",
    "A", "B", "X", "Y",
]

last = None
for payload in open("controller-data.txt", encoding="ascii"):
    payload = payload.strip()
    if not payload:
        continue

    pressed = []
    bits = int(payload[8:12], 16)
    for index, name in enumerate(KEYS):
        if name and bits & (1 << index):
            pressed.append(name)

    if int(payload[12:16], 16) > 0xFF00:
        pressed.append("LT")
    if int(payload[16:20], 16) > 0xFF00:
        pressed.append("RT")

    state = "+".join(pressed)
    if state and state != last:
        print(state)
    last = state or None
```

清理松开帧后可以分出三组输入：

```text
left left LB RB LB right left LB left
Y left right right X RT RB
Y right right left right X B left
```

与 GTA V 的 Xbox 手柄作弊码表逐项对照，三组输入分别对应：

1. `moongravity`
2. `slowmotion`
3. `drunkmode`

按题目要求使用小写并以下划线连接，得到：

```text
shellmates{moongravity_slowmotion_drunkmode}
```

## 方法总结

USB 手柄取证的核心是先确认报告布局，再把按键位图、模拟扳机和松开帧分别处理。只打印非零位图会混入重复状态；加入去重后，输入序列才能稳定地与游戏作弊码表对应。最终 flag 为 `shellmates{moongravity_slowmotion_drunkmode}`。
