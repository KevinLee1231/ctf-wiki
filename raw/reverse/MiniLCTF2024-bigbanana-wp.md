# miniLCTF 2024 bigbanana Writeup

## 题目简述

程序实现了一个栈式虚拟机。输入字符被压栈后，字节码执行加法、异或、保存中间状态和比较，只有全部比较成功才通过检查。难点不在复杂算法，而在识别 VM 指令并处理“上一轮结果参与下一轮”的状态依赖。

## 解题过程

### 还原指令语义

结合分发循环、栈变化和比较位置，可以整理出本题用到的主要操作码：

```text
10       getchar，将字符压栈
F8       压入 stack[0]
F7       压入 stack[1]
F4 imm   加立即数
01 imm   加立即数
F3       异或栈顶两个数
F2 ans   与紧随其后的常量比较
FE       检查比较结果
F0       保存本轮结果，供下一轮使用
```

第一组同时读取 `m`、`i` 两个字符，用来建立初始状态 `0x74632db5`。之后每轮只恢复一个新字符。设上一轮状态为 `last`，本轮公开的两个 F4 立即数为 $u_i,v_i$，01 立即数为 $k_i$，F2 目标为 $a_i$，则字节码对应关系可整理为：

$$((c_i+k_i)\oplus(last+u_i+v_i))=a_i.$$

每轮恢复 $c_i$ 后，再把 `last` 更新为 $c_i+k_i$。

### 逐字节求解

从字节码中顺序抽出 `f4_list`、`f01_list` 和 `answer` 后，可直接枚举可打印字符：

```python
flag = "mi"
last = 0x74632DB5

for i, target in enumerate(answer):
    for c in range(0x20, 0x7f):
        current = c + f01_list[i]
        mixed = last + f4_list[2*i] + f4_list[2*i + 1]
        if (current ^ mixed) == target:
            flag += chr(c)
            last = current
            break
    else:
        raise ValueError(f"round {i} has no printable solution")

print(flag)
```

另一种恢复方向利用最后一个字符 `}` 反推，但顺序枚举更容易逐轮校验状态。官方题解保存了完整三张常量表；当前公开仓库没有保留挑战二进制，因此无法重新从附件抽表，但按官方表运行上式得到：

```text
miniLctf{bigb4nan4_i5_v3ry_int3r5t1ng_r1ght?}
```

这里保留程序实际输出中的 `miniLctf` 大小写，不擅自改成赛事常用格式。

## 方法总结

分析小型 VM 时，先从输入、比较和输出三个边界操作向内追踪，通常只需还原真正参与校验的少量指令。题目最容易遗漏的是跨轮状态：每轮并非相互独立，求出字符后必须同步更新 `last`，否则后续全部失配。
