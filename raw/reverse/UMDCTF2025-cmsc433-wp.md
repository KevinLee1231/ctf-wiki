# UMDCTF 2025 - cmsc433

## 题目简述

程序用 C++ 模板、lambda、`std::function`、递归和 `foldr` 模仿函数式编程，
把一个本来简单的字符串校验写得十分绕。输入先被反转、删除字符并逐字节减 5，
再拆成 3 字节块；18 组 `(char, short)` 常量经 `expand` 展开后，与这些块比较。

虽然比较顺序由 `std::random_device` 驱动的 `mt19937` 随机选择，但每次都会同时删除
相同下标的 key 和输入块，所以随机性只改变检查顺序，不改变唯一正确答案。

## 解题过程

第一层检查是：

```cpp
s = reverse(s);
if (!startsWith("}eg")(s)) {
    return 0;
}
drop_str(3)(s);
map_in_place(subN(5), std::span(s));
s = reverse(s);
```

反转后的字符串必须以 `}eg` 开头，因此原输入必须以 `ge}` 结尾。删除这 3 个字符、
每字节减 5、再反转后，实际待比较字符串等价于：

```text
transformed[i] = input_without_last_3[i] - 5
```

`chunk(s, 3)` 的实现先递归处理 `s.substr(3)`，再把当前前三字节压到末尾，所以返回的
块顺序与原字符串相反。

每个 key 的展开方式为：

```cpp
std::string expand(std::pair<char, short> p) {
    char *raw = (char *)&p.second;
    return std::string{raw[0], p.first, raw[1]};
}
```

在题目使用的小端平台上，一个 `(c, n)` 变成：

```text
chr(n & 0xff) || c || chr(n >> 8)
```

因此可以直接执行逆变换：

```python
chunks = [
    chr(n & 0xff) + c + chr((n >> 8) & 0xff)
    for c, n in keys
]

transformed = "".join(reversed(chunks))
answer = "".join(chr(ord(c) + 5) for c in transformed) + "ge}"
```

18 组常量还原出的中间串为：

```text
PH?>OAv^&&ZdnZoc`ZhjnoZ_tnapi^odji\gZapi^odji\gZg\ibp\
```

逐字节加 5 并补回 `ge}` 后得到：

```text
UMDCTF{c++_is_the_most_dysfunctional_functional_language}
```

把它传给实际附件会输出 `yuor did it!`。

## 方法总结

本题的主要障碍是语法噪声，而不是复杂算法。逆向这类程序时应给每个高阶函数标注
最朴素的效果：`reverse` 是反转，`drop_str(3)` 是删前三字节，`subN(5)` 是减 5，
`chunk` 是逆序分块。还要区分“随机检查顺序”和“随机判定条件”：只要 key 与输入块
始终同步删除，随机索引不会带来爆破或概率问题。
