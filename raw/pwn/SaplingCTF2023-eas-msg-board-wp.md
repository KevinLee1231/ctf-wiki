# EAS: msg board

## 题目简述

程序用索引访问消息指针数组，但边界判断写成 index > 15 && index < 0，这两个条件不可能同时成立。负索引因此可以越界读写数组之前的 GOT 项。目标是泄漏 libc，并把 puts GOT 改成 system。

## 解题过程

根据消息数组基址与 GOT 地址计算负索引，使读取操作把 puts@got 当作消息指针，再调用 puts 泄漏其运行时地址：

~~~python
libc_base = leaked_puts - libc.sym["puts"]
system = libc_base + libc.sym["system"]
~~~

随后把普通消息槽 0 的内容设为 /bin/sh，使用另一个负索引把 puts@got 覆盖为 system。再次选择“显示消息 0”时，原本的 puts(msgs[0]) 变成 system("/bin/sh")。

取得 shell后读取：

~~~text
maple{1nvAl1D_8ouNDary_Ch3Ck}
~~~

## 方法总结

范围检查应使用 index < 0 || index >= count；错误的 && 会让整个检查恒假。把数组越界转换为 GOT 读写时，需要分别验证元素宽度、相对索引和 RELRO。题目二进制允许改写目标 GOT；若是 Full RELRO，同样思路就需要寻找其他可写函数指针。
