# What in Ternation

## 题目简述

题目校验器大量使用 AVX-512 指令。输入的每一位先被扩展到一个独立字节，装入 8 个 ZMM 寄存器；`program` 只做布尔运算、字节重排和寄存器/栈搬运。最终仅检查 `xmm0` 最低位是否为 1。

因此复杂 SIMD 代码本质上是一张 512 输入位、单输出位的布尔电路。目标不是逐条求逆，而是把指令语义翻译为 Z3 布尔表达式。

## 解题过程

### 1. 确认有效指令集合

官方解法只需处理六类核心指令：

- `VPTERNLOGD`：三输入布尔真值表；
- `VPERMB`：单源字节重排；
- `VPERMI2B` / `VPERMT2B`：从两个源按索引选字节；
- `VMOVDQA32` / `VMOVDQA64`：寄存器或栈搬运；
- `VPXOR reg,reg,reg`：清零；
- `VPBROADCASTB`：广播重排索引。

普通 `MOV`、`AND`、`SUB`、`LEAVE` 只服务函数框架，不改变被追踪的布尔数据流，可以忽略。

### 2. 建立寄存器抽象

每个 ZMM 寄存器只需要两种主要状态：

```python
RegContentBits(values=[Bool(...)] * 64)
RegContentPermute(indices=[0..127] * 64)
```

前者表示 64 个符号 bit，后者表示字节选择掩码。栈槽也保存同样的抽象对象。

初始时把 64 字节候选 flag 展开为：

```python
flag_bits = [Bool(f"flag_bit_{i}") for i in range(512)]
```

并按每 64 bit 放入 `zmm0..zmm7`。

### 3. 翻译 `VPTERNLOGD`

该指令的立即数定义三输入真值表。题目实际只使用少量立即数，例如：

```python
case 128: return A & B
case 63:  return ~(A & B)
case 254: return A | B
case 3:   return ~(A | B)
case 60:  return A ^ B
case 195: return ~(A ^ B)
case 48:  return A & ~B
case 243: return A | ~B
case 21:  return ~((A & B) | C)
case 87:  return ~((A | B) & C)
```

对寄存器的 64 个字节逐位置应用对应布尔式即可。

### 4. 翻译字节“导线”

`VPERMB` 的每个输出字节来自单个输入索引；`VPERMI2B` 与 `VPERMT2B` 的索引范围是 0 到 127，低 64 选第一个源，高 64 选第二个源：

```python
if idx < 64:
    out[i] = a[idx]
else:
    out[i] = b[idx - 64]
```

全局内存中的 64 字节常量直接作为索引数组读取。这样重排指令只重连已有 Bool 节点，不增加新的算术约束。

### 5. 求解输出 bit

符号执行到 `program` 末尾后加入：

```python
solver.add(regs[ZMM0].values[0] == True)
```

求得模型后，每 8 个 Bool 按低位优先重新拼成字符。仓库未提供官方运行 transcript，因此本文不编造具体 flag；完整可运行模型在官方 `solution/solve.py` 中，需配合题目二进制和 `iced-x86`、Z3 使用。

## 方法总结

SIMD 指令名称复杂，不代表数据流一定复杂。先确认值域后，本题所有 lane 都只有 0/1，`VPTERNLOGD` 就是逻辑门，permute 就是导线。

建模时应区分“数据寄存器”和“索引寄存器”，并严格处理 `VPERMI2B` 的双源 0..127 索引。只要这两类抽象保持正确，就不需要模拟完整 AVX-512 数值语义。
