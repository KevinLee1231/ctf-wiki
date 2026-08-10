# dyn

## 题目简述

程序在运行时生成或处理用于 memcmp 的期望 flag，静态浏览不容易直接看到完整结果。官方解法是动态调试，在 main 尾部的 memcmp 调用处检查两个参数，其中一侧已经是明文期望值。

## 解题过程

启动 GDB，在 main 中最后一次 memcmp 调用前下断点：

~~~gdb
break memcmp
run AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
~~~

System V AMD64 ABI 中，memcmp 的三个参数依次是 rdi、rsi、rdx。断下后分别检查：

~~~gdb
x/s $rdi
x/s $rsi
p/d $rdx
~~~

一侧是输入或其工作缓冲区，另一侧包含完整比较目标：

~~~text
maple{WOWGDBISPRETTYNEAT}
~~~

## 方法总结

当程序最终必须调用标准比较函数时，比较边界就是天然的观测点。动态分析不需要还原此前每一步算法，只要由调用约定读取参数并验证长度。若编译器内联了 memcmp，则应在相应比较循环或交叉引用处下断点。
