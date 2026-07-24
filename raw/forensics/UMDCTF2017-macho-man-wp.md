# MachO Man

## 题目简述

附件 `macho_man` 无法被识别为可执行文件，但开头是：

```text
19 00 00 00 48 00 00 00 5f 5f 50 41 47 45 5a 45
```

小端整数 `0x19` 是 `LC_SEGMENT_64`，后面紧跟 `__PAGEZERO`，说明文件主体仍是 64 bit Mach-O，只缺失固定长度 32 字节的 `mach_header_64`。

## 解题过程

结合现有 load command、文件布局和 x86-64 Mach-O 结构，可以恢复以下头部字段：

```text
magic       cf fa ed fe   MH_MAGIC_64
cputype     07 00 00 01   CPU_TYPE_X86_64
cpusubtype  03 00 00 80
filetype    02 00 00 00   MH_EXECUTE
ncmds       0f 00 00 00
sizeofcmds  b0 04 00 00
flags       85 00 20 00
reserved    00 00 00 00
```

把这 32 字节直接补到原文件之前：

```python
from hashlib import sha256
from pathlib import Path

header = bytes.fromhex(
    "cf fa ed fe 07 00 00 01 03 00 00 80 02 00 00 00 "
    "0f 00 00 00 b0 04 00 00 85 00 20 00 00 00 00 00"
)

recovered = header + Path("macho_man").read_bytes()
Path("macho_man.recovered").write_bytes(recovered)
print(sha256(recovered).hexdigest())
```

恢复文件的 SHA-256 为：

```text
5d77b3233dd38f68b83a776e20505f62b4f15d117e6b0b621ba6f82fc0fdb5d5
```

题目要求将该摘要放入 flag 外壳：

```text
UMDCTF-{5d77b3233dd38f68b83a776e20505f62b4f15d117e6b0b621ba6f82fc0fdb5d5}
```

整条 flag 的 SHA-256 与 README 摘要一致。

## 方法总结

文件头损坏题应先观察残留主体，而不是盲目套用 magic。`LC_SEGMENT_64` 与 `__PAGEZERO` 明确给出了格式、端序和架构；其余头部字段则可由 load command 数量及总长度交叉验证。最终摘要必须针对“补头后的完整文件”计算，不能只哈希新增头或原始残片。
