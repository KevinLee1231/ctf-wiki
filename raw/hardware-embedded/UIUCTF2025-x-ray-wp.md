# X_Ray

## 题目简述

附件的核心是约 25 MiB 的 VCD（Value Change Dump）波形，记录一个 SoC 仿真的内部信号。固件函数 `bar` 在内存中构造真实 flag，但直接打印它的 `puts_(result)` 被注释掉；随后 `rev_bar` 把同一缓冲区还原成诱饵 `uiuctf{flag_is_not_this}` 并写往 MMIO 标准输出。

因此终端输出只会给出诱饵，真实 flag 仍会在 CPU 执行 `bar`、复制缓冲区和调用 `rev_bar` 的过程中经过流水线数据总线。决定性证据是硬件仿真波形，而不是对固件静态求值本身，所以归入 Hardware/Embedded。

## 解题过程

VCD 头部为每个信号分配短标识符。题目提供的轨迹中：

```text
$scope module MEM_stage $end
$var wire 64 O+ W_data [63:0] $end
```

这对应完整路径：

```text
TOP.SOC.core.MEM_stage.W_data[63:0]
```

官方 solver 使用 `vcdvcd` 遍历该 64 位信号的全部变更，把每个值按小端序转换成 8 字节，再保留 flag 可能出现的 ASCII 字符。以下标准库版本不需要额外 VCD 包，直接复现同一逻辑：

```python
import re
import string

identifier = "O+"  # 由 VCD 头部的目标 $var 行取得
allowed = set(string.ascii_letters + string.digits + "{}_")
recovered = []

with open("trace.vcd", "rt", encoding="ascii") as f:
    for line in f:
        if not line.startswith("b"):
            continue

        bits, separator, signal = line[1:].strip().partition(" ")
        if separator != " " or signal != identifier:
            continue
        if any(bit not in "01" for bit in bits):
            continue

        value = int(bits, 2)
        chunk = value.to_bytes(8, "little")
        recovered.extend(chr(b) for b in chunk if chr(b) in allowed)

stream = "".join(recovered)
for flag in re.findall(r"uiuctf\{[A-Za-z0-9_]+\}", stream):
    print(flag)
```

提供的波形中该信号共有 2252 次已知二进制变化。ASCII 流包含寄存器中间值和大量重复字符，但正则可以直接定位：

```text
uiuctf{Ovo_fl4g_1s_this}
```

后续还能看到 `flag_is_not_this` 片段和 `HALT`，这与 `f.c` 的执行顺序一致，说明恢复的是实际流水线数据而不是从题目元数据硬编码答案。

## 方法总结

- 核心技巧：从 SoC VCD 中定位内存阶段的 64 位写数据总线，按目标端序重组每次变更并搜索结构化明文。
- 识别信号：固件在内存中短暂构造秘密但不输出，附件却提供 RTL/SoC 波形；终端诱饵与内部数据通路存在不同可见性。
- 复用要点：先从 `$scope`/`$var` 恢复完整层级和短标识符，不能只按同名 `W_data` 猜信号；还要确认位宽、端序、未知态处理和时间顺序，再用 flag 格式从噪声中验证结果。
