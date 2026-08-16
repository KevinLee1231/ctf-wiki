# HackINI2024 Serial

## 题目简述

程序要求输入用户名并生成形如 `XXXX-XXXX-XXXX` 的序列号，但主流程被 `oh_no()` 提前终止，调试时还有 `ptrace(PTRACE_TRACEME)` 检查。目标可以通过补丁绕过干扰，也可以直接静态恢复 `generate_serial()` 的计算公式。本题指定用户名为 `reverse_practitioner`。

## 解题过程

反编译可见 `total_chars()` 计算用户名全部字符的 ASCII 和，记为 $T$。`generate_serial()` 只把前 12 个字符分成三组，每组 4 个字符；第 $i$ 段为该组 ASCII 和加上 $T$：

$$
S_i=T+\sum_{j=0}^{3}\operatorname{ord}(u_{4i+j}),\qquad i\in\{0,1,2\}
$$

对于：

```text
reverse_practitioner
```

总和与三组计算如下：

```text
T = 2159
reve: 114 + 101 + 118 + 101 = 434，S0 = 2593
rse_: 114 + 115 + 101 +  95 = 425，S1 = 2584
prac: 112 + 114 +  97 +  99 = 422，S2 = 2581
```

也可以用短脚本复算：

```python
username = "reverse_practitioner"
total = sum(map(ord, username))
serial = "-".join(
    str(total + sum(map(ord, username[start:start + 4])))
    for start in range(0, 12, 4)
)
print(serial)
```

输出序列号：

```text
2593-2584-2581
```

因此 flag 为：

```text
shellmates{2593-2584-2581}
```

若选择动态补丁路线，需要绕过主函数对 `oh_no()` 的调用；`generate_serial()` 内部还把四位数字及分隔符写入仅 3 字节的临时数组，触发栈保护，因此官方补丁方案还会跳过 `__stack_chk_fail()`。静态计算不依赖这两个故意设置的故障点，更容易验证。

## 方法总结

逆向 serial 算法时，应优先抽取纯计算函数，而不是被反调试和故障分支牵着走。本题最终只涉及总字符和与三个四字符分组。通过手算和脚本得到同一结果后即可闭环；补丁法适合观察流程，但不是求出序列号的必要条件。
