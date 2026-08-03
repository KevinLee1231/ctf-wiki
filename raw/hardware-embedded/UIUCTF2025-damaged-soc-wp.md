# damaged SoC

## 题目简述

附件提供 `memory.mem`、SoC 仿真器和硬件工程。处理器是用 SystemVerilog 实现的定制 MIPS64 little-endian SoC，启动存储器中的验证签名已经损坏，需要恢复能通过自检的 39 字节 flag。

本题不能只按标准 MIPS 控制流反编译：硬件把 `syscall` 改造成了相对跳转，普通反汇编器会把真实控制流切碎。决定性信息来自 RTL 中的复位向量、异常入口和 CP0 更新逻辑，因此归入硬件与嵌入式方向。

## 解题过程

先从 SystemVerilog 实现确认三项内存约定：

- 复位后的程序计数器固定为 `0x100`；
- `memory.mem` 的前 8 字节是异常处理函数地址；
- 待验证字符串从地址 `0x8` 开始，长度必须为 39。

`.mem` 文件可以按 `@地址` 指令和每行十六进制字节恢复成平坦二进制，再以 MIPS64 little-endian 装载。类似下面的解析足够完成转换：

```python
memory = {}
address = 0

for raw in open("memory.mem", encoding="ascii"):
    line = raw.strip()
    if not line:
        continue
    if line.startswith("@"):
        address = int(line[1:], 16)
        continue
    for token in line.split():
        memory[address] = int(token, 16)
        address += 1

blob = bytes(memory.get(i, 0) for i in range(min(memory), max(memory) + 1))
open("memory.bin", "wb").write(blob)
```

接下来必须按 RTL 修复控制流。异常处理器读取 CP0 中保存的故障指令，将其右移 6 位得到 `syscall` 的 20 位 code，然后把该值加到 EPC 后 `eret`。因此这里的语义不是系统调用，而是：

$$
\mathrm{EPC}_{next}=\mathrm{EPC}_{syscall}+\left\lfloor\frac{\mathrm{instruction}}{2^6}\right\rfloor.
$$

把这些位置手工改成对应相对跳转，或在分析器中实现自定义处理后，就能看到真正的验证逻辑。验证分成几块：

1. `key[0:7]` 必须是 `uiuctf{`，末尾固定为 `abcdefghi}`；
2. 大小写和位置关系恢复 `U_Uctf`，随后固定 `m1`；
3. `key[16:28]` 经两组 XOR、循环移位、加法和交叉混合后与常量比较；
4. `key[28]` 是下划线，最后拼接固定后缀。

第三块可以直接用 8 位 BitVec 建模。关键约束如下，其中所有运算都保持原程序的 64 位或 32 位截断：

```python
from z3 import *

s = Solver()
key = [BitVec(f"key_{i}", 8) for i in range(32)]
for i in range(16, 28):
    s.add(32 <= key[i], key[i] <= 126)
s.add(key[23] + key[24] == 0x53)

chunk = Concat(*reversed(key[16:24]))
chunk2 = Concat(*reversed(key[24:28]))
v1 = 0x1337C0DE12345678 ^ chunk
v2 = 0x3EADBE3F ^ chunk2
v1 = RotateLeft(v1, 8) + 0x0123456789ABCDEF
v2 = RotateLeft(v2, 4) + 0x87654321
v1 ^= ZeroExt(32, v2) << 32
v2 ^= Extract(31, 0, v1)
v1 ^= 0xFEDCBA9876543210
v2 ^= 0x13579BDF
s.add(v1 == 0xC956B3009784E40F, v2 == 0x83C5A9D1)

assert s.check() == sat
m = s.model()
print(bytes(m.eval(key[i]).as_long() for i in range(16, 28)))
```

本地运行官方约束得到：

```text
key[16:28]: psl0ver#0d00
```

与其余直接约束合并，并送入 `SOC_run_sim` 复核，可得到：

```text
uiuctf{U_Uctf_m1psl0ver#0d00_abcdefghi}
```

## 方法总结

- 核心技巧：先从 RTL 恢复非标准 MIPS64 执行语义，再对数据校验部分做约束求解。
- 识别信号：标准反汇编出现大量无法连通的 `syscall`，而硬件工程又公开了异常与 CP0 逻辑，应优先相信硬件定义而不是通用 ISA 假设。
- 复用要点：导入裸内存时要同时确认端序、基址、复位 PC 和异常向量；Z3 中的循环移位及定宽溢出也必须与 C/RTL 完全一致。
