# upx magic1

## 题目简述

附件确实经过 UPX 压缩，但文件中的 UPX 魔数被从 `UPX!` 改成了 `UPX?`，所以直接执行 `upx -d` 会报告并非 UPX 文件。恢复三个被篡改的标记后即可正常脱壳；脱壳后的校验仍是逐字节 CRC16，但目标数组与 `upx magic 0` 不同。

## 解题过程

先在十六进制编辑器中搜索 ASCII 字符串 `UPX?`。把所有三处：

```text
55 50 58 3f    UPX?
```

改回：

```text
55 50 58 21    UPX!
```

保存为新文件后脱壳，保留原附件便于失败时重新核对：

```bash
upx -d challenge-fixed.exe -o challenge-unpacked.exe
```

分析脱壳文件，可见与上一题相同的单字节 CRC 逻辑：输入字节先左移 8 位，再迭代 8 次，多项式为 `0x1021`。新的 37 项目标数组如下，直接枚举可见 ASCII：

```python
TARGET = [
    0x8d68, 0x9d49, 0x2a12, 0xab1a, 0xcbdc, 0xb92b, 0x2e32, 0x9f59,
    0xddcd, 0x9d49, 0xa90a, 0x0e70, 0xf5cf, 0x5ed5, 0x3c03, 0x7c87,
    0x2672, 0xab1a, 0x0a50, 0x5af5, 0xff9f, 0x9f59, 0xbd0b, 0x58e5,
    0x3823, 0xbf1b, 0x78a7, 0xab1a, 0x48c4, 0xa90a, 0x2c22, 0x9f59,
    0x5cc5, 0x5ed5, 0x78a7, 0x2672, 0x5695,
]

def crc16_byte(value: int) -> int:
    crc = value << 8
    for _ in range(8):
        if crc & 0x8000:
            crc = (crc << 1) ^ 0x1021
        else:
            crc <<= 1
    return crc & 0xffff

plain = []
for expected in TARGET:
    matches = [
        value
        for value in range(0x20, 0x7f)
        if crc16_byte(value) == expected
    ]
    if len(matches) != 1:
        raise ValueError(f"{expected:#06x} 的可见字符候选为 {matches}")
    plain.append(chr(matches[0]))

print(f"hgame{{{''.join(plain)}}}")
```

输出为：

```text
hgame{noW_YOu~koNw-rea1_UPx~mAG|C_@Nd~crC16}
```

部分旧题解因为枚举范围或结果抄录问题，在 `~` 附近留下了多余字符。上面的脚本逐项检查唯一候选，37 个目标值均在可见 ASCII 中恰好命中一次，因此这里采用单个 `~` 的结果。

## 方法总结

壳识别通常依赖固定签名；修改少量 magic 字节就能让自动工具拒绝处理，但不会改变压缩数据本身。遇到“题目明确提示 UPX、工具却不识别”时，应检查段名、末尾标记和 `UPX!` 签名，再在副本上恢复。脱壳只是前置步骤，真正的输入校验仍需按程序逻辑还原。
