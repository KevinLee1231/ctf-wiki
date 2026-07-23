# What Is Memory

## 题目简述

题目给出 Windows PE，包含系统版本检查、PEB 反调试和自定义 Base64 编码。目标字符串被连续编码 10 次，并且编码器存在 off-by-one：每轮末尾字符会被 NUL 覆盖。

## 解题过程

`VerifyVersionInfoW` 和 PEB 标志只负责干扰动态分析。沿最终比较点回溯，可见程序反复调用自定义 Base64 函数：

```text
for 10 rounds:
    data = custom_base64_encode(data)
```

编码函数在写入终止符时覆盖了最后一个输出字符。因此直接用标准 Base64 逐层解码会在每轮遇到长度或填充错误。逆向时枚举当前层缺失的最后一个 Base64 字符，把候选补回后解码，并用下一层应属于 Base64 字符集、最终层应为可打印 flag 作为剪枝：

```python
import base64

alphabet = b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
candidates = [blob]

for _ in range(10):
    next_round = []
    for data in candidates:
        for tail in alphabet:
            repaired = data + bytes([tail])
            try:
                decoded = base64.b64decode(repaired, validate=True)
            except Exception:
                continue
            next_round.append(decoded)
    candidates = next_round
```

结合长度和可打印约束可得到唯一结果：

```text
UMDCTF-{H4v3_Y0U_N0T1cEd_Th4T_Th3_H3AP_L1es_T0_Y0U}
```

## 方法总结

遇到“近似标准算法”时不能强行套标准解码器。先定位实现与标准的差异，再把差异建模成有限搜索。本题每轮只丢失一个 Base64 字符，候选数可通过字符集、长度和最终格式迅速收敛；系统 API 检查与编码算法无关。
