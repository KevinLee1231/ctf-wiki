# CakeCTF 2022 luau Writeup

## 题目简述

题目入口 `main.lua` 读取 flag，并调用二进制 Lua 模块 `libflag.lua`：

```lua
if libflag.checkFlag(flag, "CakeCTF 2022") then
   print("Correct!")
else
   print("Wrong...")
end
```

校验模块保存了一组变换后的字节。公开仓库中的生成脚本表明，原 flag 先经过一系列交换，再与循环密钥 `CakeCTF 2022` 异或。目标是逆序执行这些可逆操作。

## 解题过程

生成逻辑分为两步。第一步对每个 $i$，依次与所有 $j>i$ 的位置交换：

```python
for i in range(len(flag)):
    for j in range(i + 1, len(flag)):
        flag[i], flag[j] = flag[j], flag[i]
```

第二步使用循环密钥逐字节异或：

```python
flag[i] ^= key[i % len(key)]
```

异或是自身的逆运算，所以先对密文字节再次异或密钥。交换操作的逆序必须同时反转外层和内层循环：原操作按 $(i,j)$ 的正序执行，恢复时就按完全相反的顺序再次交换同一位置。

从编译模块中恢复出的密文字节如下：

```python
enc = [
    62, 85, 25, 84, 47, 56, 118, 71, 109, 0,
    90, 71, 115, 9, 30, 58, 32, 101, 40, 20,
    66, 111, 3, 92, 119, 22, 90, 11, 119, 35,
    61, 102, 102, 115, 87, 89, 34, 34,
]
key = b"CakeCTF 2022"

for i in range(len(enc)):
    enc[i] ^= key[i % len(key)]

for i in range(len(enc) - 1, -1, -1):
    for j in range(len(enc) - 1, i, -1):
        enc[i], enc[j] = enc[j], enc[i]

print(bytes(enc).decode())
```

输出为：

```text
CakeCTF{w4n1w4n1_p4n1c_uh0uh0_g0ll1r4}
```

## 方法总结

本题的重点是准确反演变换顺序。若正向流程为“置换后异或”，逆向流程必须是“先异或，再执行逆置换”；仅仅把两段代码原样重跑并不能保证恢复。

面对 Lua/Luau 字节码时，不必先完整重建整个虚拟机。先从导出函数、常量表和最终比较逻辑提取最小必要状态，再用公开生成器或原程序验证结果，能够显著减少无关分析。
