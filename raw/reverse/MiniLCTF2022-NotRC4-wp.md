# MiniLCTF2022 NotRC4 Writeup

## 题目简述

附件是 RISC-V ELF，内部又实现了一层五指令虚拟机。题名虽然叫 NotRC4，实际校验算法更接近省略密钥扩展的 RC5：输入被视为四个 64 位字，分两组进行 12 轮加法、异或和数据相关循环左移，再与四个固定密文字比较。

## 解题过程

从虚拟机初始化函数可恢复指令表：

| opcode | 语义 |
| --- | --- |
| `0xf1` | 比较输出与目标密文 |
| `0xf2 back count` | 向前跳转并循环 `count` 次 |
| `0xf3 index` | 载入相邻两个输入字并分别加常量 |
| `0xf4 0xe1/0xe2` | 更新 `X` 或 `Y`，执行异或、数据相关旋转和加法 |
| `0xf5` | 将 `X,Y` 推入输出数组 |
| `0xff` | 结束虚拟机 |

两个常量为：

```text
k1 = 0x64627421
k2 = 0x79796473
```

加密每轮先更新 `X`，再更新 `Y`。反解必须反序执行，并让 64 位旋转的位数自然按低 6 位生效：

```python
MASK = (1 << 64) - 1

def ror64(value, bits):
    bits &= 63
    return ((value >> bits) | (value << (64 - bits))) & MASK

def decrypt_pair(x, y):
    for _ in range(12):
        y = ror64((y - k2) & MASK, x) ^ x
        x = ror64((x - k1) & MASK, y) ^ y
    return (x - k1) & MASK, (y - k2) & MASK
```

依次解密四个目标字

```text
4bc21dbb95ef82ca f57becae71b547be
80a1bdab15e7f6cd a3c793d7e1776385
```

并按目标平台小端序拼接，恢复 16 字节内部输入：

```text
I_hate_U_r1sc-V!
```

补回比赛格式得到：

```text
miniLCTF{I_hate_U_r1sc-V!}
```

## 方法总结

面对 VM 题不必先追求完整反编译器，先建立 opcode 到状态变化的最小语义表即可。数据相关旋转尤其要注意机器字宽和轮次逆序；Python 实现需显式加 $2^{64}-1$ 掩码。源码注释也给出了内部 flag，可作为解密结果的交叉验证，但 WP 中仍保留了从字节码和目标密文反推的完整路径。
