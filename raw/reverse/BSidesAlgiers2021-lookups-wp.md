# Lookups

## 题目简述

目标程序把输入每两个字节按小端序解释为一个 `uint16_t`，将其送入汇编函数 `f()`，再与 26 个固定的 32 位表项逐一比较。输入长度上限是 52 字节，因此完整目标由 26 个独立的双字节块组成。

`f()` 使用 x87 浮点指令计算：

$$
f(x)=\operatorname{float32}\!\left(x\sqrt{x}\right)
$$

返回值不是整数意义上的乘积，而是最终单精度浮点数的 32 位 IEEE 754 位模式。

## 解题过程

汇编中的关键数据流为：

```asm
fild dword ptr [rbp-12]  # 把 x 压入 x87 栈
fld st(0)                # 复制 x
fsqrt                    # 得到 sqrt(x)
fmulp                    # x * sqrt(x)
fstp dword ptr [rbp-12]  # 舍入并保存为 float32
mov eax, [rbp-12]        # 返回其原始位模式
```

每个输入块只有 16 位，直接枚举 $0\ldots65535$ 即可建立“浮点位模式 → 输入值”的反查表。恢复出的 `uint16_t` 再按小端序转回两个原始字节：

```python
#!/usr/bin/env python3
import math
import struct


targets = [
    0x4A856F35, 0x4A8D10CA, 0x4A8F1369, 0x4A9CEF82,
    0x4A9AF2DC, 0x4A81ADFA, 0x4A456E0A, 0x4A7034E9,
    0x495C0E39, 0x4A8F05AE, 0x49AD8381, 0x4A693752,
    0x4A692156, 0x4A704B1D, 0x4A86E3A2, 0x4A98F05A,
    0x4A9CEB77, 0x4A705637, 0x4A690F05, 0x4A7F68F3,
    0x4A9488C7, 0x4A7F59D7, 0x4A9CEB77, 0x49ADA321,
    0x4A8B2FB1, 0x44AEB15C,
]

wanted = set(targets)
reverse = {}

for value in range(1 << 16):
    packed = struct.pack("<f", value * math.sqrt(value))
    bits = struct.unpack("<I", packed)[0]
    if bits in wanted:
        reverse.setdefault(bits, []).append(value)

result = bytearray()
for target in targets:
    candidates = reverse[target]
    if len(candidates) != 1:
        raise RuntimeError(f"目标 {target:#x} 的候选不唯一: {candidates}")
    result.extend(candidates[0].to_bytes(2, "little"))

print(bytes(result).rstrip(b"\x00").decode())
```

最后一个双字节值是 `0x007d`，对应 `'}\x00'`，去掉字符串终止空字节后得到：

```text
shellmates{fpU_as$emb1y_s_ea5ier_than_pe0ple_th1nk}
```

源码的长度检查只有 `len > 52`，所以空输入也会跳过循环并打印成功信息；但程序不会因此输出 flag，而比赛提交值本身就是待恢复的 52 字节输入。这个逻辑绕过不能替代对目标表的反演。

## 方法总结

本题的核心是识别 x87 栈式浮点指令和单精度舍入边界。`fstp dword` 决定了比较值是 float32 位模式；如果在脚本中一直保留 Python double 而不显式打包为单精度，可能得到不同的低位结果。

当校验把输入拆成互不依赖的小块，每块域又只有 16 位时，逐块穷举往往比符号推导浮点逆函数更稳。还原时必须同时核对输入端序、块宽、浮点精度和尾随 NUL；这些表示层细节比公式 $x\sqrt{x}$ 本身更容易导致错误。
