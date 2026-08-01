# BYUCTF 2023 - xkcd 2637

## 题目简述

服务连续生成 500 道加法或乘法题。数字先写成罗马数字，再把 `I,V,X,L,C,D,M` 分别替换成 `1,5,10,50,100,500,1000`，答案也必须使用同一表示法。

## 解题过程

解析时必须按数值字符串从长到短替换，避免先处理 `1` 破坏 `10/100/1000`：

```python
def parse(s):
    s = (s.replace('1000', 'M').replace('500', 'D')
          .replace('100', 'C').replace('50', 'L')
          .replace('10', 'X').replace('5', 'V').replace('1', 'I'))
    total = 0
    value = {'I':1,'V':5,'X':10,'L':50,'C':100,'D':500,'M':1000}
    for i, ch in enumerate(s):
        total += value[ch]
        if i and value[ch] > value[s[i-1]]:
            total -= 2 * value[s[i-1]]
    return total
```

把两个操作数转整数，执行 `+` 或 `*`，再用标准贪心法转回罗马数字，最后把罗马符号映射回数字串。循环正确回答 500 次后得到：

```text
byuctf{just_over_here_testing_your_programming_skills_:)}
```

## 方法总结

这是协议自动化和表示层解析题，没有决定性的密码或漏洞原语，因此归入 `_unclassified`。稳定脚本应严格按提示符收发，并把“罗马解析、整数运算、特殊格式输出”拆成可单测函数。
