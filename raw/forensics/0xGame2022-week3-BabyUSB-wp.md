# week3BabyUSB

## 题目简述

附件是一份同时包含普通网络通信和 USB HID 键盘报告的 PCAPNG。ZIP 密码被拆成两段：一段直接藏在可追踪的通信内容中，另一段由键盘按键序列输入。恢复两段并按提示顺序拼接后即可解压得到 flag。

## 解题过程

先检查普通协议流。过滤 HTTP 报文并追踪相关 TCP 流，可以读到：

```text
Part of the zip password is s_Here
```

再过滤带 `usb.capdata` 字段的报文，并导出 HID 报告：

```bash
tshark -r 1.pcapng -Y "usb.capdata" -T fields -e usb.capdata > usbdata.txt
```

USB Boot Keyboard 报告通常为 8 字节：第 1 字节是修饰键位图，第 2 字节保留，第 3–8 字节是同时按下的键码。左、右 Shift 分别对应修饰位 `0x02`、`0x20`。下面脚本去除重复保持报告、处理 Shift 和退格，并将键码还原为文本：

```python
import string
from pathlib import Path

normal = {}
shifted = {}

for code, char in zip(range(0x04, 0x1E), string.ascii_lowercase):
    normal[code] = char
    shifted[code] = char.upper()

for code, char, shifted_char in zip(
    range(0x1E, 0x28),
    "1234567890",
    "!@#$%^&*()",
):
    normal[code] = char
    shifted[code] = shifted_char

normal.update({
    0x28: "\n", 0x2C: " ", 0x2D: "-", 0x2E: "=",
    0x2F: "[", 0x30: "]", 0x31: "\\", 0x33: ";",
    0x34: "'", 0x36: ",", 0x37: ".", 0x38: "/",
})
shifted.update({
    0x28: "\n", 0x2C: " ", 0x2D: "_", 0x2E: "+",
    0x2F: "{", 0x30: "}", 0x31: "|", 0x33: ":",
    0x34: '"', 0x36: "<", 0x37: ">", 0x38: "?",
})

output = []
previous = None

for raw_line in Path("usbdata.txt").read_text().splitlines():
    hex_report = raw_line.replace(":", "").strip()
    if len(hex_report) < 16:
        continue

    report = bytes.fromhex(hex_report[:16])
    if report == previous:
        continue
    previous = report

    modifier = report[0]
    table = shifted if modifier & 0x22 else normal

    for keycode in report[2:8]:
        if keycode == 0:
            continue
        if keycode == 0x2A:  # Backspace
            if output:
                output.pop()
            continue
        output.append(table.get(keycode, f"<0x{keycode:02x}>"))

print("".join(output))
```

键盘流还原出：

```text
Part of the password is P@33w0rD_1
```

原 PDF 的简单脚本会把保持按下的重复报告多算一次，曾显示成 `P@334w0rD_1`；按 HID 事件去重后，正确片段是 `P@33w0rD_1`。两段按顺序拼接，ZIP 密码为：

```text
P@33w0rD_1s_Here
```

解压后得到：

```text
0xGame{E43y_Traff1c_Analy3i3}
```

## 方法总结

USB 键盘流量不能只取固定位置后机械拼字符，还要理解 8 字节报告、Shift 位图、释放包、保持键造成的重复以及退格。与此同时，不应只盯住 HID；题目把另一段密码放在普通协议流中，需要先做全局协议盘点。最终应记录完整密码而不只记录两条提示，才能让 WP 独立复现。
