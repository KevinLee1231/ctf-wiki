# week1py一下

## 题目简述

附件 `1.pyc` 是 Python 2.7 字节码。反编译后可见，程序要求输入长度与 `check` 数组一致，并逐字节计算 `ord(input[i]) ^ i` 与数组比较。异或的另一操作数只是公开的字符下标，因此可以直接反推输入。

## 解题过程

先确认 PYC 版本，再用支持 Python 2.7 的反编译器：

```bash
file 1.pyc
pycdc 1.pyc
```

反编译得到的关键逻辑为：

```python
check = [
    48, 121, 101, 98, 105, 96, 125, 97, 101, 112, 115, 84, 96, 126,
    81, 123, 39, 34, 77, 42, 36, 113, 73, 39, 126, 70, 106, 108,
    114, 96, 19,
]

result = []
for i in range(len(input1)):
    result.append(ord(input1[i]) ^ i)

if check == result:
    print "right"
```

由

$$
\operatorname{check}_i=\operatorname{ord}(m_i)\oplus i
$$

可得 $m_i=\operatorname{check}_i\oplus i$：

```python
check = [
    48, 121, 101, 98, 105, 96, 125, 97, 101, 112, 115, 84, 96, 126,
    81, 123, 39, 34, 77, 42, 36, 113, 73, 39, 126, 70, 106, 108,
    114, 96, 19,
]

plain = bytes(value ^ index for index, value in enumerate(check))
print(repr(plain))
```

恢复结果是：

```text
b'0xgame{fmyy_ls_t73_90d_0f_pwn}\r'
```

最后一个字节是回车 `\r`，属于发布版校验数组中的行尾残留；实际 flag 内容为 `0xgame{fmyy_ls_t73_90d_0f_pwn}`。这也意味着原检查器在不同终端的换行处理下可能无法直接通过，静态恢复结果更可靠。

## 方法总结

- 核心技巧：反编译 PYC，并对逐字节“字符异或下标”做同样的异或逆运算。
- 识别信号：校验循环把输入字符与固定下标或固定数组组合后逐项比较。
- 复用要点：PYC 反编译器必须匹配字节码版本；恢复后还要检查 `\r`、`\n`、NUL 等不可见字节是否只是构建环境残留。
