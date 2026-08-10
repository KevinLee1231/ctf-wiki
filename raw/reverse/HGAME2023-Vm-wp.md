# Vm

## 题目简述

程序把 40 字节输入交给一台自定义虚拟机校验。真实算法没有直接出现在原生控制流里，而是由全局字节码、虚拟寄存器、数据区和虚拟栈共同实现。需要识别 VM 状态结构与指令分发器，反汇编字节码，再逆向恢复输入。

## 解题过程

### 还原虚拟机结构

`main` 先初始化一块上下文，再由 `getchar` 把输入写入全局数据区。VM 主循环持续读取 `code[ctx->ip]`，遇到 `0xff` 才退出，因此偏移 `24` 的字段可确定为指令指针。结合初始化函数、栈操作和比较指令，可以把上下文概括为：

```c
typedef struct VMContext {
    uint32_t reg[6];
    uint32_t ip;   // +0x18
    uint32_t sp;   // +0x1c
    uint32_t zf;   // +0x20
} VMContext;
```

主分发器相当于虚拟 CPU 的译码阶段。逐个进入处理函数后可得到指令表：

| 操作码 | 指令 | 作用 |
| --- | --- | --- |
| `0` | `mov` | 寄存器、立即数与 `data[reg[2]]` 之间传送 |
| `1` | `push` | 把寄存器压入虚拟栈 |
| `2` | `pop` | 从虚拟栈恢复寄存器 |
| `3` | `alu` | `add/sub/mul/xor/shl/shr` |
| `4` | `cmp` | 比较 `reg[0]` 与 `reg[1]`，更新 `zf` |
| `5` | `jmp` | 无条件跳转 |
| `6` | `jne` | `zf == 0` 时跳转 |
| `7` | `je` | `zf != 0` 时跳转 |
| `0xff` | `halt` | 结束解释执行 |

`mov` 和 `alu` 的第二个字节是子操作码，后续字节保存寄存器编号或立即数。按这些规则写反汇编器后，关键字节码被还原为：

```text
mov reg[2], 0
add reg[2], reg[3]
mov reg[0], data[reg[2]]
mov reg[1], reg[0]
mov reg[2], 50
add reg[2], reg[3]
mov reg[0], data[reg[2]]
add reg[1], reg[0]
mov reg[2], 100
add reg[2], reg[3]
mov reg[0], data[reg[2]]
xor reg[1], reg[0]
mov reg[0], 8
mov reg[2], reg[1]
shl reg[1], reg[0]
shr reg[2], reg[0]
add reg[1], reg[2]
mov reg[0], reg[1]
push reg[0]
...
pop reg[0]
mov reg[2], 150
add reg[2], reg[3]
mov reg[0], data[reg[2]]
cmp reg[0], reg[1]
```

第一段循环处理 `i = 0..39`，计算：

$$
t_i = \operatorname{swap16}\left((\operatorname{ord}(m_i)+A_i)\oplus B_i\right),
$$

其中 `swap16` 交换 16 位整数的高、低字节。结果依次压栈；第二段循环再弹栈，所以比较顺序与 `data[150:190]` 相反。

### 逆运算恢复输入

从数据区取出三组有效常量，逆序目标数组并依次撤销字节交换、异或和加法：

```python
addends = [
    0x9B, 0xA8, 0x02, 0xBC, 0xAC, 0x9C, 0xCE, 0xFA,
    0x02, 0xB9, 0xFF, 0x3A, 0x74, 0x48, 0x19, 0x69,
    0xE8, 0x03, 0xCB, 0xC9, 0xFF, 0xFC, 0x80, 0xD6,
    0x8D, 0xD7, 0x72, 0x00, 0xA7, 0x1D, 0x3D, 0x99,
    0x88, 0x99, 0xBF, 0xE8, 0x96, 0x2E, 0x5D, 0x57,
]

xors = [
    0xC9, 0xA9, 0xBD, 0x8B, 0x17, 0xC2, 0x6E, 0xF8,
    0xF5, 0x6E, 0x63, 0x63, 0xD5, 0x46, 0x5D, 0x16,
    0x98, 0x38, 0x30, 0x73, 0x38, 0xC1, 0x5E, 0xED,
    0xB0, 0x29, 0x5A, 0x18, 0x40, 0xA7, 0xFD, 0x0A,
    0x1E, 0x78, 0x8B, 0x62, 0xDB, 0x0F, 0x8F, 0x9C,
]

targets = [
    0x4800, 0xF100, 0x4000, 0x2100, 0x3501, 0x6400, 0x7801, 0xF900,
    0x1801, 0x5200, 0x2500, 0x5D01, 0x4700, 0xFD00, 0x6901, 0x5C00,
    0xAF01, 0xB200, 0xEC01, 0x5201, 0x4F01, 0x1A01, 0x5000, 0x8501,
    0xCD00, 0x2300, 0xF800, 0x0C00, 0xCF00, 0x3D01, 0x4501, 0x8200,
    0xD201, 0x2901, 0xD501, 0x0601, 0xA201, 0xDE00, 0xA601, 0xCA01,
]

flag = []
for i, value in enumerate(reversed(targets)):
    value = (((value << 8) & 0xFF00) | ((value >> 8) & 0xFF)) & 0xFFFF
    flag.append(chr((value ^ xors[i]) - addends[i]))

print("".join(flag))
```

输出为：

```text
hgame{y0ur_rever5e_sk1ll_i5_very_g0od!!}
```

## 方法总结

VM 逆向的重点不是一开始就理解整段字节码，而是先把状态结构、取指位置、操作码和寻址方式命名清楚。处理函数中出现的数据区索引、栈指针变化和条件跳转是最可靠的语义线索。完成最小反汇编器后，复杂控制流会退化成普通的逐字节变换；最后还要注意虚拟栈导致的目标数组逆序。
