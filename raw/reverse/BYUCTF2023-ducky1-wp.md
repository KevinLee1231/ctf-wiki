# BYUCTF 2023 - Ducky1

## 题目简述

附件 `inject.bin` 是 USB Rubber Ducky 的 HID 键盘注入载荷，使用默认美国键盘布局。目标是把二进制按 HID 报告解码回输入文本。

## 解题过程

标准 Ducky payload 以 8 字节键盘报告为基本单元：第 1 字节是修饰键，第 3 至 8 字节是 HID keycode。逐报告查询 US 布局，遇到 Shift 时选择大写/上档字符，忽略全零释放报告。

可使用离线 Ducky decoder，也可按上述映射自行解析。恢复的输入文本中直接出现：

```text
byuctf{this_was_just_an_intro_alright??}
```

## 方法总结

USB 键盘流不是普通字符编码；同一个 keycode 在不同布局和修饰键下会生成不同字符。解码时必须同时保留报告边界、modifier 和键盘布局。
