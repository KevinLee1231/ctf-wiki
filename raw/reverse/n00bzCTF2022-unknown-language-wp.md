# Unknown Language

## 题目简述

附件不是未知语言，而是 Python 字节码反汇编。它逐字符读取一个整数，并把 $5^{\operatorname{ord}(c)}-x$ 写入 `results.txt`。需要先还原字节码语义，再对大整数做精确逆运算。

## 解题过程

从 `LOAD_CONST 5`、`ord(flag[i])`、`BINARY_POWER` 和 `BINARY_SUBTRACT` 的栈顺序可还原核心代码：

```python
for i in range(len(flag)):
    x = input("Enter the flag: \n")
    f.write(str(5 ** ord(flag[i]) - int(x)) + "\n")
```

每轮输入已知常数，例如 `5`，即可令输出满足：

$$y_i+5=5^{\operatorname{ord}(flag_i)}.$$

官方题解使用浮点对数，但这些整数很大，浮点舍入可能把字符码算错。更稳妥的做法是在合理字符码范围内精确比较整数幂：

```python
def recover(values, submitted=5):
    out = []
    for value in values:
        target = value + submitted
        code = next(k for k in range(128) if 5 ** k == target)
        out.append(chr(code))
    return "".join(out)
```

对生成的全部结果执行恢复，得到：

```text
n00bz{rev3s1ng_w1th_l0gar1thms_wh4t?}
```

## 方法总结

字节码题应按虚拟机栈的操作数顺序还原表达式。本题还混合了可逆的指数表示；用整数幂精确搜索比浮点 `log` 更可靠，也明确利用了自己提交的已知偏移量。
