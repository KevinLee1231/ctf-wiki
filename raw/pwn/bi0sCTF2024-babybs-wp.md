# bi0sCTF 2024 - babybs

## 题目简述

附件是一个 512 字节的 x86 实模式 boot sector。程序通过 BIOS 键盘中断让用户选择 `userPassword` 的某个字节，再用上下方向键对该字节加一或减一；当 32 位值等于 `0x13371337` 时退出交互。flag 字符串直接位于 boot sector 尾部。

漏洞在于索引只做了 `input - 0x30`，没有任何范围检查。目标地址按 `userPassword + index` 计算，因此可以修改整个 16 位地址空间中相邻的代码和数据。官方 exploit 先把一段现有指令逐字节改成输出 flag 的实模式代码，再改写控制流跳转到该代码。

## 解题过程

### 确认任意相对字节修改原语

关键汇编为：

```asm
sub al, 0x30
mov [currentIndex], al
...
mov al, [currentIndex]
add ax, userPassword
mov bx, ax
add byte [bx], 1        ; 上方向键
```

下方向键路径只把最后一条换成 `sub byte [bx], 1`。`currentIndex` 是一个字节，输入字符可令它落在远超 4 字节密码缓冲区的范围。每轮可以重复发送方向键，因此任意目标字节都能通过模 256 的加减变成期望值。

构造补丁时先离线确定原字节和目标字节，再选择较短的加或减方向：

```python
def delta(old, new):
    up = (new - old) & 0xff
    down = (old - new) & 0xff
    return ("up", up) if up <= down else ("down", down)
```

### 写入实模式 payload

boot sector 以 `org 0x7c00` 装载，且 `DS=ES=SS=0`。flag 在固定偏移处，因而 payload 不需要地址泄漏。可以编写极短的 16 位例程，把 `SI` 指向 flag，循环读取字符并使用 BIOS teletype 服务 `int 0x10, AH=0x0e` 输出，遇到 `}` 或零字节停止。

官方 exploit 选择一段 8 字节代码区，将原字节

```text
8e c0 8e d8 8e c0 31 fa
```

逐字节调整为预先汇编好的短 payload 字节，然后继续用同一 OOB 原语修改附近跳转。脚本以字符 `0x35 + i` 选择各目标索引，通过方向键发送所需次数；之后选择控制流字节所在索引并修改其位移，使执行流落入新 payload。

这里不应直接照搬远程脚本中的固定延时和旧主机地址。可靠复现方式是：

1. 用 `ndisasm -b 16 -o 0x7c00 babybs.bin` 确认原字节与补丁位置；
2. 计算每个字节的模 256 最短差值；
3. 在本地 QEMU 中逐步验证补丁后的控制流；
4. 最后在远程按相同索引和次数发送按键扫描码。

当跳转被触发后，新代码直接打印 boot sector 内嵌的 flag，不需要满足正常的密码比较。

## 方法总结

本题把一个看似只能调整 4 字节密码的界面变成了相对任意字节写。实模式没有 NX、ASLR 等保护，且代码、数据和 flag 都在固定的 512 字节映像中，所以最直接的利用是修改现有代码并劫持跳转。实施时应以反汇编得到的真实偏移和字节为准，把方向键操作视为模 256 的增减写原语。
