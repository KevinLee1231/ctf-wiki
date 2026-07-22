# ez_usbpcap

## 题目简述

附件是 USB 键盘抓包。除常规字母与数字外，输入还使用了数字小键盘；很多只覆盖主键区的现成脚本会在这里漏字符。解题目标是提取 8 字节 HID Boot Keyboard 报告，正确处理修饰键、保持按键产生的重复包以及小键盘键码。

## 解题过程

先用 Tshark 导出 HID 数据。不同 Wireshark 版本可能使用 `usb.capdata` 或 `usbhid.data` 字段，可先在字段列表中确认：

```bash
tshark -r ez_usbpcap.pcapng -Y "usb.capdata" -T fields -e usb.capdata > usb.dat
```

Boot Keyboard 报告通常为 8 字节：第 1 字节是修饰键位图，第 2 字节保留，第 3～8 字节保存同时按下的键码。下面的脚本只记录相对上一报告新增的键，支持左右 Shift，并补齐 `0x59`～`0x63` 的数字小键盘映射：

```python
from pathlib import Path

normal = {0x04 + i: chr(ord("a") + i) for i in range(26)}
normal.update({0x1E + i: "1234567890"[i] for i in range(10)})
normal.update({0x59 + i: "123456789"[i] for i in range(9)})
normal.update({0x62: "0", 0x63: ".", 0x28: "\n", 0x2C: " "})
normal.update({0x2D: "-", 0x2E: "=", 0x2F: "[", 0x30: "]", 0x31: "\\"})
normal.update({0x33: ";", 0x34: "'", 0x36: ",", 0x37: ".", 0x38: "/"})

shifted = {code: value.upper() for code, value in normal.items() if value.isalpha()}
shifted.update(dict(zip(range(0x1E, 0x28), "!@#$%^&*()")))
shifted.update({0x2D: "_", 0x2E: "+", 0x2F: "{", 0x30: "}", 0x31: "|"})
shifted.update({0x33: ":", 0x34: '"', 0x36: "<", 0x37: ">", 0x38: "?"})

result = []
previous = set()

for raw_line in Path("usb.dat").read_text().splitlines():
    report = bytes.fromhex(raw_line.replace(":", "").strip())
    if len(report) != 8:
        continue

    modifier = report[0]
    current = {code for code in report[2:] if code}
    new_keys = current - previous
    table = shifted if modifier & (0x02 | 0x20) else normal

    for code in report[2:]:
        if code in new_keys:
            result.append(table.get(code, f"<0x{code:02x}>"))
    previous = current

print("".join(result))
```

还原出的输入包含：

```text
moectf{n1ha0w0y0udianl32451}
```

## 方法总结

USB 键盘流量不能只机械读取第三字节：要先确认报告格式，再处理 Shift 位图、释放包、按住不放造成的重复报告及多键同时按下。小键盘 `1`～`9` 对应 HID Usage ID `0x59`～`0x61`，`0` 与小数点对应 `0x62`、`0x63`；这正是本题相对普通键盘流量题的主要增量。
