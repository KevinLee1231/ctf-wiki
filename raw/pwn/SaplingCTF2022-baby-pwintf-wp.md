# Baby Pwintf

## 题目简述

程序把用户输入直接作为 printf 的格式字符串。随后只有当堆上的 rating 等于 0x1337 时才打印 flag。调用 printf 时，rating 指针恰好作为第 7 个可变参数可见，因此可以用 %n 把累计输出字符数写入该地址。

## 解题过程

先用一串 %p 确认参数位置，第 7 项是 rating 的地址。%n 写入“此前已输出的字符数”，所以先输出十进制 4919 个字符，再执行 <code>%7$n</code>：

~~~python
payload = b"%4919c%7$n"
io.sendlineafter(
    b"Tell me your name and I'll rate it!",
    payload,
)
~~~

$4919=0x1337$，因此 printf 返回后 rating 正好满足判断。PIE、NX 和 Full RELRO 都不影响这条数据写入，因为目标指针已经由程序主动传给 printf。服务返回：

~~~text
maple{youwe_weady_fow_the_big_boy_chawwenge}
~~~

## 方法总结

格式化字符串漏洞不仅能用 %p、%s 泄漏，也能通过 %n、%hn、%hhn 写内存。定位参数后，应根据目标值选择合适的写入宽度并处理累计字符数回绕。防御上必须使用 printf("%s", user_input)，绝不能让外部输入充当格式串。
