# Stack VM Revenge

## 题目简述

附件实现了一个基于栈的文本指令虚拟机。每个字符检查都被大量无关指令包围，并混入名为 `swapswap` 的伪混淆操作。目标是过滤没有语义影响的部分，提取每个字符对应的算术约束并逐字节逆运算。

## 解题过程

解释器表明 `swapswap` 连续交换栈顶两次，净效果是恒等操作。每个有效检查块都从 `input` 开始、到 `equal` 结束，块外的指令不影响该字符比较。过滤后，每块固定为五行：

```text
input <index>
push <A>
add | sub | xor
push <B>
equal
```

若运算为 `xor`，输入字符是 $A \oplus B$；若为 `add`，输入字符是 $B-A$；若为 `sub`，栈机计算“输入减 $A$”，所以输入字符是 $B+A$。完整提取脚本如下：

```python
with open("instructions_revenge.txt", encoding="utf-8") as handle:
    instructions = handle.readlines()

filtered = []
collect = False
for instruction in instructions:
    if "input" in instruction:
        collect = True
    if collect and "swapswap" not in instruction:
        filtered.append(instruction.strip())
    if "equal" in instruction:
        collect = False

flag = []
for i in range(0, len(filtered), 5):
    a = int(filtered[i + 1].split()[1])
    opcode = filtered[i + 2]
    b = int(filtered[i + 3].split()[1])

    if opcode == "xor":
        value = a ^ b
    elif opcode == "add":
        value = b - a
    elif opcode == "sub":
        value = b + a
    flag.append(chr(value))

print("".join(flag))
```

脚本输出：

```text
grey{one_tw0_thr33_f0ur_f1ve_s1x_s3ven}
```

把候选字符串重新输入原始 VM，程序返回 `Correct! Flag validated successfully!`。

## 方法总结

处理长 VM 指令流时，先以数据依赖确定切片边界，再删除可证明为恒等变换的混淆。这里 `input` 到 `equal` 是天然的单字符程序切片，双交换则可形式化消去；过滤完成后，问题退化为三种一字节可逆算术。
