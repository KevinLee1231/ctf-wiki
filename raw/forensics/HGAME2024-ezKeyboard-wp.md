# ezKeyboard

## 题目简述

附件是一份 USB 键盘流量。目标是从多个 USB 设备中筛出地址为 `1.2.3` 的键盘报告，按 HID Boot Keyboard 的状态变化恢复实际按键序列。该抓包比常见的 8 字节键盘报告多了一个固定的 `0x01` 头字节，并且包含组合键、长按、Caps Lock 与退格；直接套用网上只做“键码逐包映射”的脚本会产生重复字符或错误大小写。

## 解题过程

### 筛选并导出键盘报告

在 Wireshark 中使用显示过滤器：

```text
usb.src == "1.2.3" && usbhid.data
```

也可以用 `tshark` 直接导出十六进制 HID 数据：

```powershell
tshark.exe -r .\hgame_1.2.3.pcap -Y 'usb.src == "1.2.3" && usbhid.data' -T fields -e usbhid.data > hid.txt
```

官方 PDF 使用过 `usb.capdata` 字段；不同 Wireshark 版本对 HID 字段的解析名称可能不同。如果 `usbhid.data` 为空，就把上面命令中的字段改成 `usb.capdata`。判断是否导出正确的方法不是看字段名，而是确认数据中反复出现“修饰键、保留字节、六个键码槽位”的键盘报告结构。

本题每行前面额外带一个 `0x01`，去掉它后才是标准 8 字节报告：

```text
01 | modifier | 00 | key1 | key2 | key3 | key4 | key5 | key6
     \___________________ 标准 8 字节 HID 报告 __________________/
```

`modifier` 中的 `0x02` 与 `0x20` 分别表示左右 Shift，`0x01` 与 `0x10` 分别表示左右 Ctrl。后六个字节不是六次独立输入，而是“当前仍处于按下状态的键集合”，所以只有本报告中新出现的键码才应输出。

### 用状态机恢复按键

下面的脚本覆盖本题所需的字母、数字、符号和控制键，并正确处理四个容易出错的状态：

- 用当前键集合减去上一报告的键集合，消除长按造成的重复字符；
- 保留同一报告中的多个新按键，不能一包只取一个键码；
- Caps Lock 只在按键上升沿翻转，字母大小写由 `Shift XOR Caps Lock` 决定；
- 遇到 Backspace 立即删除已经恢复的最后一个字符。

```python
from pathlib import Path


KEYMAP = {
    0x28: ("\n", "\n"),
    0x29: ("", ""),          # Esc
    0x2A: ("\b", "\b"),      # Backspace
    0x2B: ("\t", "\t"),
    0x2C: (" ", " "),
    0x2D: ("-", "_"),
    0x2E: ("=", "+"),
    0x2F: ("[", "{"),
    0x30: ("]", "}"),
    0x31: ("\\", "|"),
    0x32: ("#", "~"),        # Non-US # and ~
    0x33: (";", ":"),
    0x34: ("'", '"'),
    0x35: ("`", "~"),
    0x36: (",", "<"),
    0x37: (".", ">"),
    0x38: ("/", "?"),
}

for code in range(0x04, 0x1E):
    lower = chr(ord("a") + code - 0x04)
    KEYMAP[code] = (lower, lower.upper())

for code, normal, shifted in zip(
    range(0x1E, 0x28),
    "1234567890",
    "!@#$%^&*()",
):
    KEYMAP[code] = (normal, shifted)


def parse_report(text: str) -> bytes | None:
    raw = bytes.fromhex(text.strip().replace(":", ""))
    if len(raw) >= 9 and raw[0] == 0x01 and raw[2] == 0x00:
        raw = raw[1:9]        # 去掉本题特有的 0x01 头
    elif len(raw) >= 8:
        raw = raw[:8]
    else:
        return None
    return raw


output: list[str] = []
previous: set[int] = set()
caps_lock = False
stop = False

for line in Path("hid.txt").read_text().splitlines():
    if not line.strip():
        continue

    report = parse_report(line)
    if report is None:
        continue

    modifier, reserved, *slots = report
    if reserved != 0:
        continue

    current = {code for code in slots if code != 0}
    new_codes = [code for code in slots if code != 0 and code not in previous]
    shift = bool(modifier & 0x22)
    ctrl = bool(modifier & 0x11)

    for code in new_codes:
        if code == 0x39:      # Caps Lock：只在上升沿翻转一次
            caps_lock = not caps_lock
            continue

        if ctrl and code == 0x06:  # Ctrl-C，flag 已经输入完毕
            stop = True
            break

        if code == 0x2A:
            if output:
                output.pop()
            continue

        pair = KEYMAP.get(code)
        if pair is None:
            continue

        if 0x04 <= code <= 0x1D:
            use_shifted = shift ^ caps_lock
        else:
            use_shifted = shift
        output.append(pair[int(use_shifted)])

    previous = current
    if stop:
        break

print("".join(output))
```

运行后恢复的输入末尾还带有一次 `Ctrl-C`，去掉控制操作后，flag 为：

```text
hgame{keYb0a1d_gam0__15_s0_f0n__!!~~~~}
```

官方 PDF 说明了错误脚本的几个典型问题，但没有展示最终字符串。抓包特有的 `0x01` 头、Caps Lock 上升沿处理和最终输出与 [Mantle 的 HGAME 2024 Week 4 复现](https://csmantle.top/2024/02/29/ctf-writeup-hgame2024-week4.html#ezKeyboard) 交叉核对；关键逻辑已完整写入上面的状态机，无需依赖外链才能复现。

## 方法总结

- USB 键盘报告描述的是当前按键状态，不是逐包字符流；恢复文本时必须比较相邻报告的键集合。
- 长按、同时按键、左右 Shift、Caps Lock 与 Backspace 都会破坏简单的“键码查表”脚本，应显式维护状态。
- 厂商或抓包层可能在标准报告前增加 Report ID。本题固定头为 `0x01`，应先通过保留字节和报告长度验证，再去掉该字节。
- 网上现成脚本只能作为键码表参考；对状态机行为应结合原始报告自行验证。
