# Pop Quiz

## 题目简述

远程服务连续给出七类基础数论问题，包括取模、模幂、普通幂以及离散对数。每题都有时限，适合用脚本解析数字并调用对应运算，而不是手工反复计算。

## 解题过程

连接后按提示文本识别题型：

- $x\bmod p$ 直接用 Python 的 %；
- $x^y$ 使用整数幂；
- $x^y\bmod p$ 使用 pow(x, y, p)，避免先生成巨大整数；
- 给定 $g^x\equiv h\pmod p$ 时，用离散对数算法求 $x$。

一个简化的处理框架如下：

~~~python
from sympy.ntheory import discrete_log

if "modular exponent" in prompt:
    answer = pow(x, y, p)
elif "discrete log" in prompt:
    answer = discrete_log(p, h, g)
elif "mod" in prompt:
    answer = x % p
else:
    answer = x ** y
~~~

每轮把十进制结果和换行发回，完成七题后得到：

~~~text
maple{0nt0-7h3-n3xt!}
~~~

## 方法总结

本题考查的是把文字题稳定映射到运算。模幂应始终使用三参数 pow；离散对数不能误写成普通对数。自动化时先从一轮完整交互中确认数字顺序和题型关键词，再写解析器，可避免把底数、结果和模数位置读反。
