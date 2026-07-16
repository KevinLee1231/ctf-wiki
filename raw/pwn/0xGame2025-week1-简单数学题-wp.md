# 简单数学题

## 题目简述

程序连续生成 1000 道随机整数加、减、乘题。只有全部回答正确才会执行 `system("/bin/sh")`。这里没有必要利用内存破坏，主要障碍是稳定解析输出并自动完成大量交互。

## 解题过程

### 关键逻辑

源码中的 `get()` 同时生成题面与正确答案，主循环比较输入结果；任意一次错误都会退出：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/ioctl.h>

void* buf;

void init()
{
    setvbuf(stdin, 0);
    setvbuf(stdout, 0);
    setvbuf(stderr, 0);
}

int get(int n,int m,int sym)
{
    memset(buf,0,0x100);
    switch(sym)
    {
        case 0:
            sprintf(buf,"%d %s %d = ?\n",n,"+",m);
            return n+m;
        case 1:
            sprintf(buf,"%d %s %d = ?\n",n,"-",m);
            return n-m;
        case 2:
            sprintf(buf,"%d %s %d = ?\n",n,"x",m);
            return n*m;
    }
}

int main()
{
    init();
    buf=(void *)malloc(0x100);
    srand(time(0));
    int n,m,num,sym;
    puts("Are you good at math?");
    puts("Kore wa shiren da!");
    for(int i=0;i<1000;i++)
    {
        n=rand()%1000;
        m=rand()%1000;
        sym=rand()%3;
        int ret=get(n,m,sym);
        printf(buf);
        scanf("%d",&num);
        if(num!=ret)
            exit(0);
        puts("Good work!");
    }
    puts("Congratulations on completing the challenge");
    system("/bin/sh");
    return 0;
}
```

每行格式固定为 `<n> <op> <m> = ?`。直接拆分字段并建立运算符映射，比对服务端文本调用 `eval()` 更安全，也避免乘号 `x` 不是 Python 运算符的问题。

```python
from pwn import *
import operator

io = process('./pwn')
ops = {
    '+': operator.add,
    '-': operator.sub,
    'x': operator.mul,
}

io.recvuntil(b"Kore wa shiren da!\n")
for _ in range(1000):
    line = io.recvuntil(b" = ?\n").splitlines()[-1].decode()
    left, op, right, _, _ = line.split()
    answer = ops[op](int(left), int(right))
    io.sendline(str(answer).encode())
    io.recvuntil(b"Good work!\n")

io.interactive()
```

完成第 1000 轮后程序进入 `/bin/sh`，此时切换到交互模式即可读取 flag。

## 方法总结

- 核心技巧：为固定文本协议编写同步可靠的自动化交互脚本。
- 识别信号：大量重复问答、答案计算简单、全部通过后给 Shell。
- 复用要点：按明确分隔符接收数据，不依赖脆弱切片；对有限运算符使用白名单映射，避免直接执行未经验证的表达式。
