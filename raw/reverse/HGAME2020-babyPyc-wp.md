# babyPyc

## 题目简述

附件是 Python 3.7 的 `.pyc`，并通过修改字节码跳转偏移干扰普通反汇编。真正的校验逻辑还藏在嵌套列表推导式的 code object 中。恢复逻辑后可知：程序把 36 字节 flag 主体反转、转置成 $6\times6$ 矩阵，再做相邻行累加和 Base64 比较。

## 解题过程

Python 3.7 的 PYC 头为 16 字节。跳过文件头后用 `marshal` 读取顶层 code object：

```python
import dis
import marshal

with open("task.cpython-37.pyc", "rb") as file:
    file.read(16)
    code = marshal.load(file)

dis.dis(code)
for const in code.co_consts:
    if hasattr(const, "co_code"):
        print(const.co_name)
        dis.dis(const)
```

不能只看顶层输出；列表推导式会编译成独立 code object，并继续嵌套在 `co_consts` 中。把这些对象递归展开后，关键逻辑为：

```python
raw_flag = flag[6:-1][::-1]
ciphers = [[raw_flag[6 * row + col] for row in range(6)]
           for col in range(6)]

for row in range(5):
    for col in range(6):
        ciphers[row][col] = (ciphers[row][col] + ciphers[row + 1][col]) % 256
```

目标 Base64 为：

```text
Qp+ng3SeWoXClJN4cYm3frO8n5rIqL/Nrreuks7JR1JPM19w
```

加法从第 0 行依次进行，故逆运算要从第 4 行倒序，用当前行减下一行。随后按列取回原序列并再次反转：

```python
from base64 import b64decode

cipher = b64decode(
    "Qp+ng3SeWoXClJN4cYm3frO8n5rIqL/Nrreuks7JR1JPM19w"
)
matrix = [
    [cipher[6 * row + col] for col in range(6)]
    for row in range(6)
]

for row in range(4, -1, -1):
    for col in range(6):
        matrix[row][col] = (matrix[row][col] - matrix[row + 1][col]) % 256

message = b"".join(
    bytes(matrix[row][col] for row in range(6))
    for col in range(6)
)[::-1]
print(f"hgame{{{message.decode()}}}")
```

输出：

```text
hgame{pYtH0n_oPc0D3_I5_$O_iNt3Re5T1nGg89!!}
```

## 方法总结

- 核心技巧：按正确 Python 版本解析 PYC，并递归检查 `co_consts` 中的嵌套 code object。
- 识别信号：反汇编行号或跳转明显异常，但 code object 仍能被 `marshal` 解析时，可能只是通过偏移干扰阅读。
- 复用要点：矩阵累加属于有方向的原地变换，逆运算必须反向遍历；转置和最后的字符串反转也不能遗漏。
