# UMDCTF2022 How to Breakdance Writeup

## 题目简述

题目给出 `how_to_breakdance.pcapng`，要求恢复用户输入的 YouTube 密码。流量不是普通网络会话，而是 USB 键盘的 HID 报告；决定性步骤是从 USB 完成事件中提取按键 Usage ID，再按键盘映射还原字符。

## 解题过程

先在 Wireshark 中检查 USB 数据包，可见变化字段为 `usb.capdata`。只保留设备完成传输的记录并导出该字段：

```bash
tshark -r how_to_breakdance.pcapng \
  -Y "usb.urb_type == URB_COMPLETE" \
  -T fields -e usb.capdata > hex_keyboard_inputs
```

标准键盘 HID 报告通常为 8 字节：第 1 字节是修饰键，第 2 字节保留，后 6 字节是同时按下的键。该流量只需要小写字母、数字和下划线，因此把非零 Usage ID 按如下范围映射即可：

```text
0x04..0x1d -> a..z
0x1e..0x27 -> 1..0
0x2c       -> 空格
0x2d       -> _
```

仓库的官方脚本逐行读取十六进制报告，并使用同一映射打印非零键值。按捕获顺序拼接后得到密码：

```text
1_luv_70_f1nd_c7f_fl46s
```

题目要求在密码外包裹比赛前缀，最终 flag 为：

```text
UMDCTF{1_luv_70_f1nd_c7f_fl46s}
```

## 方法总结

USB 键盘流量的重点是识别 HID 报告结构，而不是把 `usb.capdata` 当作 ASCII。实际题目若包含大写字母、组合键或长按，还需要处理修饰键、按下与释放事件以及重复报告；本题输入简单，按报告顺序提取非零 Usage ID 即可恢复完整密码。
