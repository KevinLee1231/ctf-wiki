# week1_re3

## 题目简述

题目是标准 Base64 编码逆向。64 位 PE 读取输入后调用编码函数，把结果与硬编码的 56 字符 Base64 串比较。编码表是标准：

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

因此直接提取目标串并 Base64 解码即可。

## 解题过程

Ghidra 搜索 `please input your flag:` 定位主函数，核心流程为：

```text
scanf input
FUN_1400010e0(input, output)  # Base64 encode
compare output with DAT_1400032a0
```

`DAT_1400032a0` 以 `int` 数组形式存储，每个 4 字节整数的低字节是一个 Base64 字符。提取得到：

```text
ZmxhZ3tkYWJkZDMzMC0yN2RiLTRlOGEtYjBjZi0zMTAzMDVlYzAzNWJ9
```

解码：

```python
import base64

s = "ZmxhZ3tkYWJkZDMzMC0yN2RiLTRlOGEtYjBjZi0zMTAzMDVlYzAzNWJ9"
print(base64.b64decode(s).decode())
```

输出：

```text
flag{dabdd330-27db-4e8a-b0cf-310305ec035b}
```

## 方法总结

- 核心技巧：识别标准 Base64 编码函数并逆向目标串。
- 识别信号：64 字符标准表、3 字节到 4 字符、`=` padding 处理同时出现时基本可确认 Base64。
- 复用要点：目标数组若按 `int` 存储，要只取低字节，否则会混入空字节导致解码失败。
