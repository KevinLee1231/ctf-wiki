# Hackergame2020 超精巧的数字论证器 WP

## 题目简述

服务端连续给出 32 个小于 114514 的自然数。每次要提交一个 Python 2 整数表达式，使表达式的值等于目标数，同时删除所有运算符和括号后，剩余数字必须恰好是 `114514`，总长度不超过 256 字符。

这是受限表达式构造题，核心是整数恒等式与十进制展开，不属于密码或逆向，暂归 `_unclassified`。

## 解题过程

直接枚举二元运算符只能覆盖一部分数。关键恒等式是：

$$
-\mathord{\sim}x=x+1,
$$

因为按位取反满足 $\mathord{\sim}x=-x-1$。因此在任何子表达式前增加一次 `-~`，都能在不引入新数字的情况下把值加一；反向的 `~-` 则减一。

目标最多六位，可以按十进制 Horner 形式构造：

$$
n=(((((d_1\times10+d_2)\times10+d_3)\times10+d_4)\times10+d_5)\times10+d_6).
$$

表达式中六个数字必须依次使用 `1、1、4、5、1、4`。第一个 `1` 通过 `~-1` 变成 0；其余五个数字分别在前面重复若干 `-~`，都可变成 10。每处理一位目标十进制数字，就先乘以对应的 10，再用若干 `-~` 加上该位。

```python
def get_answer(number):
    base = 10
    source_digits = "114514"
    terms = []

    # ~-1 == 0，作为 Horner 展开的初值。
    first = source_digits[0]
    terms.append("~-" * int(first) + first)

    # 把余下每个原始数字递增到 10。
    for char in source_digits[1:]:
        terms.append("-~" * (base - int(char)) + char)

    answer = ""
    for index, term in enumerate(terms):
        digit = (number // base ** (len(terms) - 1 - index)) % base
        if index == 0:
            answer = term
        else:
            answer = f"({answer}*{term})"
        answer = "-~" * digit + answer

    assert eval(answer) == number
    assert "".join(c for c in answer if c.isdigit()) == "114514"
    assert len(answer) <= 256
    return answer
```

连接服务后，每次读取等号左侧的目标数，调用 `get_answer()` 并提交；连续 32 次通过即可取得用户对应的 flag。这个构造覆盖从 0 到 114513 的全部目标，不依赖随机搜索。

## 方法总结

表达式长度受限时，逐次加一不可行，但 `-~` 提供了“无新数字的常数增量”，再配合十进制 Horner 展开就能把增量次数限制在每位最多 9 次。构造后同时断言数值、数字串和长度三个条件，可以在联网提交前发现优先级或括号错误。
