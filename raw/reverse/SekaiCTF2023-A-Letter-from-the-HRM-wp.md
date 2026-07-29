# A Letter from the Human Resource Management

## 题目简述

附件 `instructions.txt` 看起来像一段陌生汇编，包含 `INBOX`、`OUTBOX`、`COPYFROM`、`COPYTO`、`BUMPUP`、`BUMPDN`、`JUMPZ` 等指令，末尾还有多段 Base64 风格的 `DEFINE LABEL` 和 `DEFINE COMMENT`。这些特征对应游戏 Human Resource Machine 的汇编语言和关卡标签格式。

题面中的 “worker”“42 years”“7 billion”“floor”“big check mark” 都是在提示该游戏。解码注释后还会得到输入约束：flag 以 ASCII 数值进入收件箱，字符范围是 33 至 126。

## 解题过程

### 解码地板标签

程序把常量画在地板标签上，标签分别使用多种记数系统。无需保留外部对照图，只要将关键映射转写成数值：

| 地板索引 | 表示法 | 数值 |
|---:|---|---:|
| 0, 1, 2 | 罗马数字 `CXX`、`CXXI`、`LVI` | 120, 121, 56 |
| 3, 4, 5 | 东阿拉伯数字 `٣٢`、`١٠٧`、`٨٩` | 32, 107, 89 |
| 6, 7, 8 | 盲文数字 | 77, 103, 78 |
| 9, 10, 11 | 天城文数字 | 73, 126, 125 |
| 12, 13, 14 | 希腊数字 | 10, 3, 24 |
| 18 | 中文数字 | 5 |
| 21, 22 | 计数器初值与零 | 14, 0 |
| 23, 24 | 对号与叉号 | 成功、失败 |

由此得到比较表：

```python
floor = {
    0: 120, 1: 121, 2: 56, 3: 32, 4: 107,
    5: 89, 6: 77, 7: 103, 8: 78, 9: 73,
    10: 126, 11: 125, 12: 10, 13: 3, 14: 24,
    18: 5, 21: 14, 22: 0,
}
```

### 还原程序逻辑

Human Resource Machine 没有原生 XOR，指令 29 至 77 使用加减、倍增和符号跳转逐位模拟异或。外层循环的计数器位于地板 21，初值为 14；地板 18 保存乘数 5。

每轮开头先把计数器加一，再重复加 5，因此生成的异或键依次为：

```text
75, 70, 65, 60, ..., 10, 5
```

设当前计数器为 $n$、输入字符 ASCII 为 $x$，程序计算：

$$
y = x \oplus 5(n+1)
$$

随后用间接寻址 `SUB [21]` 将 $y$ 与 `floor[floor[21]]` 比较。若不相等，输出地板 24 的叉号并结束；若相等，计数器减一并检查下一个字符。全部 15 个字符通过后，输出地板 23 的对号。

等价伪代码为：

```python
counter = 14

while counter >= 0:
    key = 5 * (counter + 1)
    value = inbox()

    if (value ^ key) != floor[counter]:
        outbox("cross")
        break

    counter -= 1
else:
    outbox("check")
```

### 直接反推 flag

XOR 可逆，所以第 $i$ 个输入字符无需爆破：

$$
x = \text{floor}[n] \oplus 5(n+1)
$$

按计数器从 14 递减到 0 生成字符：

```python
floor = [
    120, 121, 56, 32, 107,
    89, 77, 103, 78, 73,
    126, 125, 10, 3, 24,
]

flag = "".join(
    chr(floor[counter] ^ (5 * (counter + 1)))
    for counter in range(14, -1, -1)
)
print(flag)
```

输出：

```text
SEKAI{cOnGr47s}
```

将该字符串逐字符送入解释器时，最后会输出对号，验证了数值标签和程序语义均还原正确。

## 方法总结

- 核心技巧：先识别 Human Resource Machine 指令集，再把图形标签还原成地板常量，最终将冗长的手工 XOR 汇编化简为一行公式。
- 识别信号：`INBOX`/`OUTBOX`、地板间接寻址、`BUMPUP`/`BUMPDN` 以及图形化标签共同指向游戏式自定义解释器。
- 复用要点：逆向陌生 VM 或 esolang 时，不必逐条照抄执行。先确定状态对象、循环变量、比较点和成功输出，再把实现层的算术过程归纳为高层运算，通常能直接反解输入。
