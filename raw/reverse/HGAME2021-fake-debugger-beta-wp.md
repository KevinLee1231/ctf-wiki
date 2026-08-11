# fake_debugger beta

## 题目简述

`challenge` 用文本界面模拟单步调试：每按一次空格，就显示 `eax`、`ebx`、`ecx` 和 `zf`，再比较当前输入字节。附件同时提供原始 `test.py`，其中已经保留异或数组和目标数组，因此不需要真的逐步交互，直接逆运算即可恢复 24 字节 flag。

## 解题过程

每轮执行的关键逻辑是：

```python
eax = ord(user_input[index]) ^ arr[index]
ebx = check[index]
if eax != ebx:
    print("Wrong Flag! Try again!")
```

所以：

$$
\text{input}_i=\text{check}_i\oplus\text{arr}_i
$$

附件中的两个数组为：

```python
arr = [
    23, 45, 67, 89, 13, 24, 35, 46, 57, 35, 46, 57,
    37, 48, 38, 13, 16, 37, 58, 63, 41, 73, 52, 94,
]
check = [
    127, 74, 34, 52, 104, 99, 122, 65, 76, 124, 101, 87,
    21, 71, 121, 105, 117, 71, 79, 120, 78, 122, 70, 35,
]

flag = bytes(a ^ b for a, b in zip(arr, check))
print(flag.decode())
```

输出：

```text
hgame{You_Kn0w_debuGg3r}
```

## 方法总结

模拟调试器中的寄存器名只是对 Python 变量的包装，真正的校验仍是逐字节异或。看到附件同时给出参考程序时，应先还原其数据流，再决定是否需要自动化交互；异或等式任意两项已知即可求第三项，单步输出只用于帮助观察，不是必须完成的障碍。
