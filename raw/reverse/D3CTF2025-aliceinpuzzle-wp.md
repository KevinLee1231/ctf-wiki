# AliceInPuzzle

## 题目简述

题目是 Linux arm64 程序。外层父进程充当调试器，用 `ptrace` 附加子进程并处理 `SIGSTOP` / `SIGTRAP`；子进程通过实时信号把输入指针交给父进程，父进程会反转输入字符串并辅助解 SMC。内层 payload 通过 `memfd_create` 写入内存文件，再用 `fexecve` 执行，最终校验一个 Fillomino 棋盘答案并对编码结果做 MD5。

## 解题过程

题目是 arm64 下 linux，开个虚拟机跑起来大概长这样：可以按照这个输出顺下来流程

```text
$ ./main
[Kei] Searching for Alice...
[Kei] There you are.
[Kei] So, she wants to play a game with you...
[Kei] But don't expect to meet her directly.
[Kei] I'll be the one watching. Always.
[Kei] Hmm...
[Kei] Hmm...
[Alice] Hey there, brave challenger~
[Alice] I'm Alice, and you've just stepped into my little puzzle world!
[Alice] The board is all set up and ready to go... Are you feeling ready to take on the challenge?
[Alice] Now, show me your solution when you're ready!
> {aaaaaaaaa}
[Kei] I see... A secret message from Alice?
[Kei] Hmm...
[Alice] Hmm... that doesn't seem quite right...
[Alice] Ah... can't you play more seriously?
```

因为题目外层没有去符号，应该也能看个大概出来是怎么样的。一开始父进程先ptrace ATTACH 子进程上去（一开始 Kei 说的前五行话），之后是一系列状态同步检查和错误处理（都可以不看）。

之后是 ptrace 主循环，里面 if 检查的是 ptrace 返回的 status，根据常量查找知道对应的意义：如果子进程产生 SIGSTOP 信号并且 has_pending_input==true，那么触发 mangle 函数。如果子进程踩到 brk 0 了（触发 SIGTRAP 信号）就触发handle_sigtrap 函数，开始解 SMC。mangle 函数功能是将字符串反转写回：

```text
父进程 ptrace 主循环：
- `SIGSTOP && has_pending_input == true`：调用 `mangle`，把 `pending_ptr` 指向的输入字符串反转后写回子进程。
- `SIGTRAP`：调用 `handle_sigtrap`，读取子进程 PC 附近的指令并推进 SMC 解密。
```

其参数来自于之前注册的信号处理函数：

```text
实时信号处理函数会把 `has_pending_input` 置为 true，并从信号附带信息中取出输入字符串指针保存到 `pending_ptr`。
```

而信号处理函数里面刚好有把 has_pending_input 赋值为 true 的语句，那么这个逻辑就说得通了：子进程的信号被父进程捕获了，这里作为父进程的信号处理函数执行。因为这个是“实时信号”，里面有指针信息能够传递给父进程，这个指针就是反转的地址。所以子进程中抛出信号所带的指针就是需要反转的字符串指针。handle_sigtrap 函数则是继续完成这道题的关键（也许是题眼罢）

查找 ptrace 的使用方法可以知道，这里是先获得寄存器信息。参考链接 `https://stackoverflow.com/questions/70334705/how-would-a-debugger-running-in-linux-windows-read-the-pc-register-on-arm32-aa` 的关键信息是：在 ARM Linux 上读取被调试进程寄存器通常通过 `ptrace(PTRACE_GETREGSET, ...)` 配合 `iovec` 完成，返回的寄存器结构可按平台 ABI 解析，其中 PC 寄存器给出当前执行地址。本题利用这一点读取 tracee 的 pc，然后读取 pc 所在位置内存；如果前 32 字节是 `0xD4200000`，就加上一个 delta，直到下一次读取到的是 `0xD4200000` 停下来。

所以这个玩意就是一个调试器版本的 SMC。回到 tracee_main 函数，这个函数调用了一个 syscall。

memfd_create 是在内存中创建一个虚拟文件的系统调用，这里给它起了个名字并获取fd。然后写文件，通过 fexecve 执行这个虚拟文件。（有一些函数需要进去出来才能正确显示参数）。所以提取出 payload 出来，可以发现也是一个 elf 文件。根据上面的分析，找主函数要搜索 brk 0（0xD4200000），应该有两个地方，低地址的那个才是。之后如果符号不好看的话可以先把外层的符号提取出来，做成 sig 文件然后给内层用，这样静态的标准库函数基本都能看了。然后写一个脚本把中间的 SMC恢复一下，就能看到子进程的主逻辑了。子进程进去的第一件事是注册了一个信号处理函数，其实没什么用，直接 NOP 掉就行。之后是解压缩 puzzle，大概长这样：

```c
int vb_decode(const uint8_t *input, int input_len, int32_t *output) {
    int out_index = 0;
    int32_t num = 0;
    int shift = 0;
    for (int i = 0; i < input_len; i++) {
        uint8_t byte = input[i];
        num |= (byte & 0x7F) << shift;
        if ((byte & 0x80) == 0) {
            output[out_index++] = num;
            num = 0;
            shift = 0;
        } else {
            shift += 7;
        }
    }
    return out_index;
}
```

是一个解压缩数字的算法，因为数字可能会比较小，不用 32 位也能装得下，所以有压缩的空间。至于装不装得下用最高位来表示。将子程序弄出来之后，是可以把所有有关 ptrace 的东西全部 patch 掉开始调试弄明白逻辑的。中间一大坨大概就是输入，然后 strchr 确定大括号里面的内容（检查格式），然后又是一大坨复制出来到 v32这里后面开始就是关键函数了：用 linux_eabi_syscall_w 发送一个实时信号，这里就和父进程交互了（就是那个 secret message，把指针给传进去了）。父进程最开始也注册了一个信号处理函数，里面收到信号会把 has_pending_input 设置成 True，然后从信号里获取到一个指针 pending_ptr（就是输入的东西）。之后子进程自己停下来，父进程收到暂停，在主循环里面把这玩意反转了（mangle 函数），再塞回去原来的位置（PTRACE_POKEDATA）。随后 `sub_401D00` 将 hex 转为 byte，再进入 Fillomino 棋盘合法性检查。

然后和上面一样解压缩出来，并测试是否合法（是不是和已经填入数字冲突了）。如果冲突，Alice 会输出 `Ah... can't you play more seriously?`，意思是“不能更认真一点玩吗？”；如果没有冲突，则对每个格子调用一次 `sub_401BD0`。这里的游戏叫 Fillomino，规则是相同数字的块连在一起称为一个区域，每个区域的块数必须等于每个块的数字，有在线求解器可以使用。检验逻辑如下：

```c
bool visited[BOARD_SIZE] = {};
constexpr bool is_valid(int x, int y) {
    return x >= 0 && x < 9 && y >= 0 && y < 9;
}
// DFS 检查连通区域大小
int dfs(int x, int y, int8_t val, const int8_t *b) {
    int count = 1;
    visited[x * 9 + y] = true;
    int dx[4] = {0, 1, 0, -1};
    int dy[4] = {1, 0, -1, 0};
    for (int dir = 0; dir < 4; dir++) {
        int nx = x + dx[dir];
        int ny = y + dy[dir];
        if (!is_valid(nx, ny))
            continue;
        int idx = nx * 9 + ny;
        if (!visited[idx] && b[idx] == val) {
            count += dfs(nx, ny, val, b);
        }
    }
    return count;
}
// Main validation loop
bool is_correct = true;
for (int i = 0; i < 9; i++) {
    for (int j = 0; j < 9; j++) {
        int idx = i * 9 + j;
        if (input[idx] == 0 || visited[idx])
            continue;
        int8_t val = input[idx];
        int region_size = dfs(i, j, val, input);
        if (region_size != val) {
            is_correct = false;
        }
    }
}
```

解题脚本如下，最后要记得 md5：

```python
import struct
# 答案棋盘，81 个 int8_t
board = [
    3, 6, 6, 6, 6, 6, 6, 2, 2,
    3, 3, 2, 2, 4, 4, 3, 3, 1,
    6, 6, 6, 6, 5, 4, 4, 3, 6,
    6, 2, 2, 6, 5, 6, 6, 6, 6,
    4, 7, 7, 7, 5, 5, 5, 8, 6,
    4, 4, 4, 7, 7, 6, 3, 8, 8,
    5, 6, 6, 6, 7, 6, 3, 3, 8,
    5, 6, 6, 6, 7, 6, 6, 8, 8,
    5, 5, 5, 2, 2, 6, 6, 8, 8
]
# 将每 4 个 int8_t 合并为一个 int32_t，然后进行 vb_encode
def vb_encode(num):
    output = []
    while num >= 128 or num < 0:
        output.append((num & 0x7F) | 0x80)
        num >>= 7
    output.append(num & 0x7F)
    return output
encoded = []
# 如果不能整除4，末尾补零
if len(board) % 4 != 0:
    board += [0] * (4 - (len(board) % 4))
for i in range(0, len(board), 4):
    chunk = board[i:i+4]
    num = struct.unpack('<i', bytes(chunk))[0]  # 小端序解为 int32
    encoded.extend(vb_encode(num))
# 转换为十六进制字符串 反转
flag = ''.join(f"{byte:02x}" for byte in encoded)[::-1]
# md5 哈希
import hashlib
hash_object = hashlib.md5(flag.encode())
flag = hash_object.hexdigest()
print(flag)
```

flag: d3ctf{a6410f9a866c52763c11bce9fb8b06ca}

## 方法总结

- 核心技巧：把父进程视为自定义调试器，沿 `ptrace` 状态机理解 SMC、实时信号传参和输入反转，再从内存 `memfd` payload 中恢复真实校验。
- 识别信号：Linux 逆向中出现 `ptrace ATTACH`、实时信号、`brk 0` / `SIGTRAP` 和 `memfd_create + fexecve` 时，应怀疑多进程调试器式保护和内存 ELF。
- 复用要点：先提取并修复 SMC 后的内层逻辑，再把最终游戏规则还原成标准 puzzle 求解；本题 Fillomino 校验要求每个连通块大小等于块内数字。
