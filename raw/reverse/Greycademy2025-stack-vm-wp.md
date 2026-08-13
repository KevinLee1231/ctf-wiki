# stack-vm

## 题目简述

附件包含一个 Python 栈虚拟机和 `instructions.txt`。VM 支持 `push`、`input`、`xor`、`add`、`sub` 与 `equal`；验证程序由重复的五条指令组成，每组独立约束 flag 的一个字符。

## 解题过程

一组典型指令为：

```text
input 0
push 105
xor
push 14
equal
```

执行顺序是把 `input[0]` 和常数 105 压栈、异或，再要求结果等于 14。因此该字符满足 `input[0] = 105 XOR 14`。所有组结构完全一致，可以每五行恢复一个字符：

```python
with open("instructions.txt", "r", encoding="utf-8") as file:
    instructions = file.readlines()

flag = []
for offset in range(0, len(instructions), 5):
    key = int(instructions[offset + 1].split()[1])
    expected = int(instructions[offset + 3].split()[1])
    flag.append(chr(key ^ expected))

print("".join(flag))
```

脚本输出：

```text
grey{i_l0ve_st4ck_th4_fl3gs!!!}
```

再把结果交给原始 `stack_vm.py`，VM 输出 `Correct! Flag validated successfully!`，完成正向验证。

## 方法总结

逆向小型 VM 时先恢复每条 opcode 的栈效果，再寻找指令流的重复基本块。本题每个字符之间没有状态耦合，`equal` 失败即退出，所以不需要符号执行或暴力搜索，只需逐块反演异或。最终 flag 为 `grey{i_l0ve_st4ck_th4_fl3gs!!!}`。
