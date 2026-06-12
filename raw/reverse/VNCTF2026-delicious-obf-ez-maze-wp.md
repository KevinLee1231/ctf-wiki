# Delicious obf & ez_maze

## 题目简述

本文件对应 VNCTF2026 Reverse 方向的两道题：`delicious obf` 和 `ez_maze`，根据 [VNCTF2026 出题笔记](https://singlehorn.github.io/2026/02/04/VNCTF2026%E5%87%BA%E9%A2%98%E7%AC%94%E8%AE%B0/) 补全题目机制和解法主线。

`delicious obf` 是一个 Windows x64 逆向题。程序的 flag 校验函数被控制流混淆打散，典型混淆块形如：

```asm
lea  r10, [rip + label_pad]
mov  r11d, imm1
xor  r11d, imm2        ; 结果固定为 4
add  r10, r11
call $5
add  rsp, 8
push r10
ret
```

目标地址前还会放 4 字节垃圾数据，实际执行时通过 `+4` 跳过垃圾字节。程序中还存在 `jz/jnz` 变体，本质仍然是把真实目标地址计算到 `r10` 后再 `ret` 跳转。

校验逻辑还叠加了反调试和 SMC。程序读取 `gs:[0x60]` 处的 PEB `BeingDebugged` 标志；未被调试时，会修改伪装成 GCC 版本字符串后方的 16 字节 key。关键关系可以抽象为：

```c
is_debug = *(uint8_t *)(gs:[0x60] + 2);
if (!is_debug) {
    for (int i = 0x50; i < 0x60; i++) {
        padding[i] ^= 0x10;   // 源码中为 0x1919810，低 8 位生效
    }
}
```

也就是说，`padding` 表面上像编译器字符串，实际与 key 相邻，`padding + 0x50` 正好落到 key 区域。另一个函数会检测 `int3` 断点字节 `0xCC`，再对 `loc_140001977` 做 SMC 修复，这也是调试时容易看到乱码函数的原因。

`ez_maze` 是一个带魔改 UPX 壳和入口花指令的 MFC 程序。脱壳后可通过 MFC 控件消息定位到 `OnCommand`，主要逻辑在对应脱壳程序的 `0x1920` 附近。核心题目机制是 20x20 迷宫：程序使用 `srand(100)` 固定随机种子，从 `(0,0)` 通过 DFS 挖通迷宫，终点为 `(19,19)`。解题重点不是手走迷宫，而是还原生成算法、求最短路，再按程序里被改动过的 `WASD` 键位映射转换成实际输入。

## 解题过程

### delicious obf

先从字符串定位到主校验函数。直接反编译会失败，是因为真实指令被切成很多小块，每块末尾都通过 `call $5; add rsp, 8; push r10; ret` 完成间接跳转。这个结构不改变语义，只是阻断反编译器的线性控制流恢复。

去混淆时不需要逐块手工跟踪。对每个 `lea r10, [rip + disp32]` 开头的混淆块，目标地址可以按下面的方式恢复：

```text
target = current_addr + 7 + disp32 + 4
jmp_rel32 = target - (current_addr + 5)
```

其中 `7` 是 `lea r10, [rip + disp32]` 的长度，`+4` 是跳过目标处的 4 字节垃圾数据，最后减去 `5` 是因为要把整段替换成 5 字节 `jmp rel32`。修复策略是：

1. 从 `lea` 到 `ret` 整段 NOP。
2. 目标地址前的 4 字节垃圾数据 NOP。
3. 在原混淆块开头写入 `E9 <rel32>`。
4. 对 `jz/jnz` 变体采用同样的目标计算方式，只是原块长度不同。

自动 patch 后，IDA 可能仍把部分代码块识别成独立函数。此时需要删除误识别函数，再把这些块作为 function tail 并回主函数，否则伪代码里仍然会断裂。

控制流恢复后，主校验链大致是：

1. 输入先经过 `type_change`。
2. 程序把输入复制到 `buf1`，再对 `buf1` 做改版 TEA。
3. `memcmp` 附近存在花指令，但最终返回值并不是 `memcmp` 的结果，而是 SMC 修复后的 `loc_140001977`。

这里有两个容易误判的点。第一，TEA 操作的是输入副本，不能只盯着原始 input。第二，`loc_140001977` 初看是乱码函数，真正原因是程序把软件断点 `0xCC` 当作触发信号，用 SMC 修复该函数；如果只静态看原始函数体，会误以为还有一层未知混淆。

最后按调试得到的密文和 key 写逆运算。这个 TEA 变体的关键差异是：`sum` 在处理 4 个 64 位块时没有每块重置，而是跨块连续变化。下面是等价 Python 解密脚本：

```python
data = [
    0x738EA1B9, 0xF5B06584, 0xDCF952D5, 0x6FC28041,
    0x1DA40CF1, 0x07572A62, 0xB4C49903, 0x9BA536D8,
]
key = [0xF9B2917F, 0x2A9D0847, 0x0C874A13, 0xA0253AD3]

mask = 0xffffffff
delta = 0x61C88647
sum_ = (32 * delta * 4) & mask

for i in range(6, -1, -2):
    a, b = data[i], data[i + 1]
    for _ in range(32):
        a = (a + ((b + (((16 * b) & mask) ^ (b >> 5))) ^ ((key[sum_ & 3] + sum_) & mask))) & mask
        sum_ = (sum_ - delta) & mask
        b = (b + ((a + (((16 * a) & mask) ^ (a >> 5))) ^ ((key[(sum_ >> 11) & 3] + sum_) & mask))) & mask
    data[i], data[i + 1] = a, b

plain = bytes((x >> (8 * j)) & 0xff for x in data for j in range(4))
print(plain.decode())
```

脚本可恢复出 `VNCTF{N0w_Y0u_Kn0w_SMC_4nd_@bf!}`，用作算法和 key 恢复正确性的验证。

### ez_maze

先查壳能看到它是魔改 UPX。用 x64dbg 打开后，入口点附近存在花指令，单步可以找到接近原始入口的位置；即使开头的 `push` 一类指令被 NOP 掉，仍可用常规 ESP/RSP 定律定位 OEP，再 dump 并修复导入表。

脱壳后按 MFC 程序处理。可以用 xspy 找控件消息，再跟 `OnCommand`。出题人笔记中提到 `OnCommand` 有两个偏移，在脱壳后的程序里跟到 `0x1920` 附近即可看到主要迷宫逻辑。

迷宫生成逻辑可以抽象成下面的过程：

```text
srand(100)
map[20][20] = wall
stack = [(0, 0)]
map[0][0] = road
dirs = [(2,0), (-2,0), (0,2), (0,-2)]

while stack is not empty:
    cx, cy = stack[-1]
    valid = all unvisited cells two steps away
    if valid:
        choose one by rand() % len(valid)
        open the wall cell between current and chosen cell
        open the chosen cell
        push chosen cell
    else:
        pop stack

map[19][19] = road
open one neighboring cell of the end point
```

因为随机种子固定，迷宫也是固定的。直接复刻生成算法后，对可走格子跑 BFS 即可得到逻辑方向路径。根据出题人笔记中给出的迷宫结果，最短逻辑路径可压缩表示为：

```text
D2 R8 D4 L2 D2 L2 D2 R4 D2 R6 U4 R2 D4 R2 D7 R
```

展开后是：

```text
DDRRRRRRRRDDDDLLDDLLDDRRRRDDRRRRRRUUUURRDDDDRRDDDDDDDR
```

注意这只是“上下左右”的逻辑路径，不一定能直接作为键盘输入。题目修改了 `WASD` 键位映射，因此最终输入前还要回到 `OnCommand` 的按键分支，把逻辑方向转换成程序实际接受的字符。

## 方法总结

`delicious obf` 的核心是把两类干扰分开处理：先识别 `call $5 + push target + ret` 这类等价跳转，把控制流恢复成正常 `jmp`；再处理 SMC 和反调试，不要被表面上的乱码函数或伪装字符串误导。遇到 `padding + offset` 修改 key 的写法时，要检查全局变量在内存中的相邻关系，而不是只看变量名。

`ez_maze` 的核心是常规壳和 GUI 逆向流程的组合：先脱壳恢复可分析程序，再用 MFC 消息定位按钮/按键处理函数，最后把迷宫生成算法同构出来求路。固定种子迷宫不要手工试走，直接生成图并 BFS；如果题目改了按键映射，路径和输入字符要分开处理。
