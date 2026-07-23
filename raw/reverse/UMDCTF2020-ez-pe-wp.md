# EZ-PE

## 题目简述

附件是被故意破坏多个头字段的 64 位 Windows PE。程序主体仍在，只要修复 DOS 签名、PE 签名、机器类型、文件特征和可选头魔数，就能让加载器正常解析并运行。

## 解题过程

对照 PE32+ 结构可以确定五处错误：

| 文件偏移 | 原值 | 修复值 | 含义 |
|---:|---:|---:|---|
| `0x000` | `13 37` | `4d 5a` | DOS `MZ` 签名 |
| `0x100` | `4d 5a 00 00` | `50 45 00 00` | `PE\0\0` 签名 |
| `0x104` | `00 02` | `64 86` | AMD64 机器类型 `0x8664` |
| `0x116` | `22 20` | `22 00` | 文件特征 `0x0022` |
| `0x118` | `0b 01` | `0b 02` | PE32+ 魔数 `0x020b` |

可以按小端序直接修补：

```python
from pathlib import Path

data = bytearray(Path("easy_PEasy.exe").read_bytes())
data[0x000:0x002] = b"MZ"
data[0x100:0x104] = b"PE\x00\x00"
data[0x104:0x106] = (0x8664).to_bytes(2, "little")
data[0x116:0x118] = (0x0022).to_bytes(2, "little")
data[0x118:0x11a] = (0x020b).to_bytes(2, "little")
Path("easy_PEasy-fixed.exe").write_bytes(data)
```

修复后的程序正常运行并输出：

```text
UMDCTF-{Y0u_@re_truly_th3_PE_M@$t3r}
```

README 中的哈希与修复后程序的实际输出不一致，本文以可执行文件运行结果为准。

## 方法总结

修复损坏 PE 时应从结构层级逐步验证：先签名，再 COFF 头和可选头，最后才看节表与导入。机器类型与可选头位数必须一致，否则即使签名正确，系统加载器仍会拒绝文件。
