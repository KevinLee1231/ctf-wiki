# week2N1k0la's love

## 题目简述

官方 WP 仅确认本题存在格式化字符串漏洞：程序把用户输入直接作为 `printf` 一类函数的格式串，因此既能用 `%p`/`%s` 读取栈和内存，也能用 `%n` 系列向指定地址写入计数值。

当前公开仓库不包含本题二进制，原 PDF 也明确未提供 EXP，所以无法从现有证据准确恢复格式串参数偏移、保护配置、目标地址和最终写入值。下面保留的是可复现的定位流程，而不是虚构挑战专属常量。

## 解题过程

先发送一组带标记的指针输出，查找 `0x41414141...` 位于第几个参数：

```text
AAAA.%1$p.%2$p.%3$p.%4$p.%5$p.%6$p.%7$p.%8$p
```

也可以让 pwntools 为本地附件自动测量 offset：

```python
from pwn import *

def execute_fmt(payload):
    io = process("./main")
    io.sendline(payload)
    return io.recvall()

finder = FmtStr(execute_fmt)
print(finder.offset)
```

随后用 `checksec` 和反汇编确定实际写入目标：Partial RELRO 时通常检查可写 GOT 项和程序中的后门/重入函数；Full RELRO 时则需要结合栈地址或 libc 泄露选择返回地址等其他目标。确定 `offset`、`target` 和 `value` 后，可生成短写 payload：

```python
payload = fmtstr_payload(
    offset,
    {target: value},
    write_size="short",
)
```

发送前还要确认目标程序是一次性读取还是可重复交互、是否会在写入前输出额外字符，以及地址中是否包含输入通道禁止的坏字节；这些都会影响 `%n` 的实际计数。

## 方法总结

格式化字符串利用的固定步骤是“确认输入成为格式串 → 定位参数偏移 → 泄露所需地址 → 根据保护选择可写控制目标 → 用 `%n` 分段写入”。现有公开材料不足以给出本题最终 EXP；若重新取得原二进制，应按上述流程补回实测 offset、目标符号和最终 flag，而不能把模板常量冒充题目结论。
