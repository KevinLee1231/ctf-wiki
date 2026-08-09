# PyJail 2

## 题目简述

第二版把输入限制为不超过 50 个字符，并新增 import、os、=、txt、read、dict、分号、冒号、flag、subprocess、write、input、下划线等黑名单。然而 open、print、列表推导式和字符串连接仍可用，敏感文件名可以在运行时拼接。

## 解题过程

检查发生在 Python 执行前，只搜索原始输入字符串。因此把 flag.txt 拆成几个不含 flag 或 txt 的字面量，运行时再由加法连接：

~~~python
print([l.strip() for l in open("f"+"lag.tx"+"t")])
~~~

该表达式满足长度限制，也没有出现连续的 flag、txt、read 或下划线。open 返回的文件对象可直接迭代，列表推导式读取每一行，strip 去掉换行，print 把全部内容送到输出。

flag.txt 中混有大量填充行，找到符合赛事格式的一行：

~~~text
maple{pyth0n_0n3_lInerz_UwU}
~~~

仓库 ctfd.json 错误复用了另一道题的 flag；hosted/flag.txt 才是部署时被读取的真实证据。

## 方法总结

黑名单既挡不住运行时字符串拼接，也挡不住文件对象的迭代协议。限制字符数只会促使攻击者寻找更短的语法。判断 jail 是否安全时应沿对象能力分析，而不是统计禁词数量；只要 open 和输出仍可达，读取本地秘密就已经成立。
