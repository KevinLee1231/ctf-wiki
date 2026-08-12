# DownUnderCTF 2020 - In a pickle

## 题目简述

附件 `data` 是 Python pickle 序列化数据。可见字符串暗示其中保存了 flag，但 flag 的主体被拆成字典里的整数。直接对未知 pickle 使用 `pickle.load()` 可能执行构造对象时携带的任意代码，因此先用标准库 `pickletools` 检查 opcode，再从这份仅含基础类型的流中还原字典。

## 解题过程

先做无执行的结构检查：

```python
from pathlib import Path
from pickletools import genops

raw = Path("data").read_bytes()
ops = list(genops(raw))

dangerous = {
    "GLOBAL", "STACK_GLOBAL", "REDUCE", "NEWOBJ", "NEWOBJ_EX",
    "OBJ", "INST", "EXT1", "EXT2", "EXT4", "PERSID", "BINPERSID",
}
print([(op.name, arg) for op, arg, _ in ops if op.name in dangerous])
print(sorted({op.name for op, _, _ in ops}))
```

本题文件共有 81 个 opcode，使用的是文本协议；集合只有 `MARK`、`DICT`、`INT`、`STRING`、`PUT`、`SETITEM`、`STOP`，危险 opcode 列表为空。完整反汇编可用：

```bash
python -m pickletools data
```

从 `INT`、`STRING` 和后续 `SETITEM` 可直接读出字典，不需要执行 pickle。键 `1` 到 `3` 分别对应 `D`、`UCTF`、`{`，键 `4` 到 `22` 是十进制字符码，键 `23` 是 `}`。核心条目为：

```text
4:112  5:49   6:99   7:107  8:108  9:51
10:95  11:121 12:48  13:117 14:82  15:95
16:109 17:51  18:53  19:53  20:52  21:103 22:51
```

对键 `4` 到 `22` 的数值按顺序应用 `chr()`：

```python
values = [112, 49, 99, 107, 108, 51, 95, 121, 48, 117,
          82, 95, 109, 51, 53, 53, 52, 103, 51]
flag = "DUCTF{" + ''.join(map(chr, values)) + "}"
print(flag)
```

得到：

```text
DUCTF{p1ckl3_y0uR_m3554g3}
```

## 方法总结

pickle 是可执行的对象构造格式，不能把“只是数据文件”当作安全保证。处理未知 pickle 时，应先用 `pickletools.dis` 或 `pickletools.genops` 审计 `GLOBAL`、`REDUCE` 等可导致代码执行的 opcode；像本题这样只含基础类型时，可以直接从 opcode 参数恢复数据。题目的第二层只是按键序把十进制 ASCII 码转回字符。
