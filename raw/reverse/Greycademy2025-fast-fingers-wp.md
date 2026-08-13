# fast-fingers

## 题目简述

程序是一个 ncurses 打字游戏：内置 500 字符文章，每 15 ms 只读取一次按键。成功条件不仅要求字符正确，还要求恰好在第 1 至 500 个 tick 各记录一个对应字符；正常人工输入几乎不可能满足。

## 解题过程

`verify_perfect` 对日志施加三项约束：`log_count == 500`、`ticks_spent == 500`，以及第 `i` 项必须同时满足 `tick == i + 1`、`ch == PASSAGE[i]`。成功日志随后按“大端 4 字节 tick + 1 字节字符”串联并计算 SHA-256，摘要与 `ENCRYPTED_FLAG` 异或得到 flag。

一种直接做法是在 `getch` 返回后下断点，把返回寄存器改为下一个正确字符。官方未剥离符号二进制对应的 GDB 脚本核心为：

```gdb
file ./fast-fingers
break *0x4025d9
commands
    silent
    set $eax = PASSAGE[cursor_pos]
    continue
end
run
```

也可以不运行 TUI，直接按验证函数构造唯一的理想日志并重算解密值：

```python
import hashlib
import struct

# passage 是从程序的 PASSAGE 常量恢复出的 500 字符字符串
log = b"".join(
    struct.pack(">I", index + 1) + char.encode()
    for index, char in enumerate(passage)
)
digest = hashlib.sha256(log).digest()

encrypted = bytes([
    0x49, 0x8e, 0x71, 0x03, 0xe2, 0x2a, 0xfe, 0x32,
    0xe4, 0x86, 0xbc, 0x1f, 0x83, 0xc9, 0x22, 0xdd,
    0x66, 0x13, 0x37, 0x47, 0xcd, 0x6c, 0xd8, 0xfd,
    0x68, 0x9e, 0xcc, 0xc8,
])
print(bytes(value ^ digest[i] for i, value in enumerate(encrypted)).decode())
```

本地从源码常量解析出的 `passage` 长度确认为 500，重算结果为 `grey{fattest_fingers_ar0und}`。

## 方法总结

这题的关键不是提高输入速度，而是恢复“成功日志”这一内部状态。动态方法修改 `getch` 返回值，静态方法直接重建日志和哈希；两条路径都必须保留 tick 的大端序和从 1 开始的编号。最终 flag 为 `grey{fattest_fingers_ar0und}`。
