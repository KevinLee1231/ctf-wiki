# UMDCTF 2019 - Anti

## 题目简述

附件是一个 Windows PE。程序先调用 `IsDebuggerPresent` 做反调试检查，再验证一个 14 字符命令行参数，最后按固定下标重排输入生成 flag。

## 解题过程

静态分析可避开反调试分支。程序内嵌的一组十六进制字节解码为：

```text
y2cjav5te-k7ju
```

校验逻辑会把每个字节减去 2，因此正确参数是：

```text
w0ah_t3rc+i5hs
```

通过校验后，程序按以下下标顺序从参数中取字符：

```python
password = "w0ah_t3rc+i5hs"
order = [
    6, 0, 1, 2, 3, 4, 5, 3, 6, 7, 6, 4, 8, 9,
    9, 4, 0, 3, 2, 5, 4, 10, 11, 4, 5, 3, 10, 13,
]
body = "".join(password[index] for index in order)
print(f"UMDCTF-{{{body}}}")
```

输出：

```text
UMDCTF-{3w0ah_th3r3_c++_what_i5_this}
```

该字符串的 SHA-256 与官方摘要一致。

## 方法总结

反调试只影响动态观察，不会改变核心算法。优先定位输入长度、比较常量和成功分支，可以直接还原校验；随后把字符重排逻辑写成短脚本复算，比人工抄写更不易出错。最终还应以官方摘要验证下标和大小写。
