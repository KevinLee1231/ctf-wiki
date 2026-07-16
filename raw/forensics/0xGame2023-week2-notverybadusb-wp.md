# notverybadusb

## 题目简述

PCAPNG 记录了 BadUSB 模拟键盘输入的 USB HID 报告。目标是恢复键盘命令、分析其下载的 PowerShell 脚本，再计算脚本所下载软件的 MD5。题目报告比标准 8 字节键盘报告多一个前导字节，解析前必须去掉。

## 解题过程

在 Wireshark 显示过滤器中输入 `usb.src == "2.8.4"`，可将流量收敛到该键盘设备的中断传输。随后应导出每个包的 `usb.capdata` 字段，而不是把 Wireshark 菜单中的“导出特定分组”误当作字段导出；用 `tshark` 可以确定性完成这一步：

```bash
tshark -r notverybadusb.pcapng -Y 'usb.src == "2.8.4"' \
  -T fields -e usb.capdata > usbdata.txt
```

标准键盘报告的第 0 字节是修饰键，第 2 字节是首个按键码。本题每行解码后为 9 字节，先删除第 0 个额外字节，再按 HID usage 表转换：

```python
letters = "abcdefghijklmnopqrstuvwxyz"
digits = "1234567890"
shift_digits = "!@#$%^&*()"
punct = {
    0x28: ("\n", "\n"), 0x2B: ("\t", "\t"), 0x2C: (" ", " "),
    0x2D: ("-", "_"), 0x2E: ("=", "+"), 0x2F: ("[", "{"),
    0x30: ("]", "}"), 0x31: ("\\", "|"), 0x33: (";", ":"),
    0x34: ("'", '"'), 0x35: ("`", "~"), 0x36: (",", "<"),
    0x37: (".", ">"), 0x38: ("/", "?"),
}

out = []
for line in open("usbdata.txt", encoding="ascii"):
    text = line.strip().replace(":", "")
    if not text:
        continue
    report = bytes.fromhex(text)
    if len(report) == 9:
        report = report[1:]  # 题目特有的额外前导字节
    if len(report) != 8:
        continue

    modifier, code = report[0], report[2]
    if code == 0:
        continue
    shifted = bool(modifier & 0x22)  # left/right Shift

    if 0x04 <= code <= 0x1D:
        ch = letters[code - 0x04]
        out.append(ch.upper() if shifted else ch)
    elif 0x1E <= code <= 0x27:
        i = code - 0x1E
        out.append(shift_digits[i] if shifted else digits[i])
    elif code == 0x2A and out:
        out.pop()  # Backspace
    elif code in punct:
        out.append(punct[code][int(shifted)])

print("".join(out))
```

恢复的命令隐藏启动 PowerShell，并从 `hxxp://zysgmzb[.]club/hello/notveryevil.ps1` 下载执行脚本。该地址已作失活处理，不应直接访问。随 WP 保存的脚本内容表明，它把官方星穹铁道安装程序下载为桌面上的 `evil.exe` 并启动；原下载域为 `autopatchcn[.]bhsr[.]com`。

对在隔离环境中取得的样本计算 MD5：

```powershell
(Get-FileHash -Algorithm MD5 -LiteralPath "./evil.exe").Hash.ToLower()
```

结果为 `ece22dea2b0c6c7f3857164344ad94b4`，按题目要求提交：

```text
0xGame{ece22dea2b0c6c7f3857164344ad94b4}
```

## 方法总结

USB 键盘取证要先锁定设备和端点，再核对报告长度、修饰键、按键码与释放帧。后续载荷应静态分析并对 URL 失活处理，不在宿主机直接执行；哈希必须针对脚本实际下载的文件，而不是 PowerShell 文本或 PCAP 附件。
