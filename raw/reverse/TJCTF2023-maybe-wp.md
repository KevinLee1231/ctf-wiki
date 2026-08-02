# maybe

## 题目简述

程序内置 28 字节数组，并对输入的第 $i$ 个字符检查：

$$
\text{guess}[i]\oplus\text{guess}[i-4]=\text{enc}[i-4],\qquad 4\le i<32.
$$

已知 flag 前四字节是 `tjct`，后续字符可以按间隔 4 的递推直接恢复。

## 解题过程

从 `tjct` 作为种子，对每个位置计算：

$$
\text{guess}[i]=\text{enc}[i-4]\oplus\text{guess}[i-4].
$$

```python
enc = b"\x12\x11\x00\x15\x0bH<\x12\x0cD\x00\x10Q\x19.\x16\x03\x1cB\x11\x0aJrV\x0dztO"
flag = list("tjct" + "_" * 28)

for i in range(4, 32):
    flag[i] = chr(enc[i - 4] ^ ord(flag[i - 4]))

result = "".join(flag)
print(result)
```

输出为：

```text
tjctf{cam3_saw_c0nqu3r3d98A24B5}
```

将结果送回程序时，长度和全部 XOR 条件均通过。

## 方法总结

- 固定距离 XOR 关系会把字符串分成若干独立链；只要每条链的首字符已知，就能线性恢复后续内容。
- flag 固定前缀往往正是递推所需的初始状态，应先检查约束依赖距离。
- 恢复后应重新执行原比较循环，而不是只凭可打印性判断答案。
