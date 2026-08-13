# Artifact 4

## 题目简述

附件是一个 Python 实现的玩具虚拟机。它把 25 字节用户输入放在内存 `1000..1024`，把有重复字符的 key 放在 `2000..2024`，再执行自定义 opcode；每次比较失败都会退出，全部通过才执行 `Correct`。核心是恢复 VM 指令语义和比较索引。

## 解题过程

VM 的重要 opcode 可以归纳为：

```text
0: push register
1: pop register
2: register = register + immediate
3: register = memory[register + offset]
4: memory[register + offset] = register
5: compare two registers
6: assert comparison result
7: success
8: call
```

入口反复设置 `X` 和 `Y`、调用地址 0 的比较函数并断言 `A`。比较函数的实际语义是：

```python
def compare(i, j):
    return user_input[i] == key[j]
```

从指令序列抽出 25 个 key 索引后直接重排：

```python
key = "____aadeghhimnorrstttvy{}"
order = [
    8, 15, 7, 22, 23, 21, 12, 0, 11, 17,
    1, 13, 14, 18, 2, 19, 9, 4, 20, 3,
    10, 5, 16, 6, 24,
]
flag = "".join(key[i] for i in order)
print(flag)
```

输出为 `grey{vm_is_not_that_hard}`，将其交给原 VM 后得到 `Correct!`。

## 方法总结

逆向自定义 VM 时，先找成功/失败 opcode，再围绕它恢复最小数据流，不必完整模拟所有状态。本题的重复模式揭示了“输入索引对 key 索引”的约束；把 dispatcher 还原成少量 load、store、compare、call 语义后，长指令表就退化为一次字符重排。
