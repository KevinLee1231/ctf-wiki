# PacMan

## 题目简述

题目只给出一句提示：

> Let's play Pac-Man..

目标是一个未加密的 iOS IPA，主程序为 64 位 ARM64 Mach-O `MachActorVM`。应用界面实现了一个 25×13 的 Pac-Man 迷宫，但游戏只是外层：玩家吃到 1000 颗豆子后，程序才进入真正的 flag 生成逻辑。该逻辑把 72 个静态 actor 通过 4 个 Mach port worker 串成状态机，校验完成后再用最终状态作为 RC4 密钥解密 40 字节密文。

题目的决定性障碍不是 iOS 权限、签名或组件漏洞，而是恢复 Mach-O 中的游戏状态机、Mach actor 协议和最终解密算法，因此按普通逆向题处理。

## 解题过程

### 1. 解包与初步定性

IPA 本质上是 ZIP。解包后，主要分析对象是：

```text
Payload/MachActorVM.app/MachActorVM
```

`strings` 能看到以下高价值线索：

```text
Pac-Man
VM UNLOCKED
score:%06u/%06u  beans:%05u  moves:%05u
frida
frida-server
frida-agent
openssh
sshd
dropbear
cycript
objection
substrate
substitute
ellekit
libhooker
jailbreakd
```

这说明程序同时包含游戏逻辑、隐藏 VM 和反调试/反越狱检查。

用 IDA 打开 `MachActorVM` 后，可以识别出 Objective-C 类 `ViewController`，UI 层的调用关系如下：

| 函数 | 地址 | 作用 |
|---|---:|---|
| `-[ViewController restartGame]` | `0x100005088` | 重置全局游戏状态并生成首帧 |
| `-[ViewController directionUp]` | `0x100005128` | 调用 `stepGame:0` |
| `-[ViewController directionDown]` | `0x100005130` | 调用 `stepGame:1` |
| `-[ViewController directionLeft]` | `0x100005138` | 调用 `stepGame:2` |
| `-[ViewController directionRight]` | `0x100005140` | 调用 `stepGame:3` |
| `-[ViewController stepGame:]` | `0x100005148` | 执行一步游戏并刷新界面 |
| `-[ViewController updateWithFrame:]` | `0x1000051B0` | 显示地图；解锁后调用 flag 生成器 |

### 2. 外层 Pac-Man 游戏

`restartGame` 的核心调用为：

```c
sub_100007048();        // 初始化状态
sub_1000073E4(frame);   // 生成初始画面
updateWithFrame(frame);
```

`sub_100007048` 把 Pac-Man 坐标初始化为 `(1, 1)`，清零分数、豆子数和移动次数，再调用十次 `sub_1000071C0`。后者在迷宫的可通行位置随机放置 10 颗豆子，并保证豆子之间不重叠。

每次按方向键都会进入 `sub_100007A68`。方向到坐标增量的映射为：

```text
0: up     ( 0, -1)
1: down   ( 0, +1)
2: left   (-1,  0)
3: right  (+1,  0)
```

程序使用 `#` 表示墙。移动后如果命中某个豆子槽位，就执行：

```c
score += 10;
beans += 1;
respawn_that_bean_slot();
```

所以地图上始终最多只有 10 颗豆子，却必须不断寻找重生豆子。真正的解锁条件是：

```c
if ((unsigned int)score >> 4 >= 0x271)
    unlocked = 1;
```

`0x271` 是 625，因此最低需要 `score >= 10000`。每颗豆子只有 10 分，也就是需要吃 1000 颗豆子。渲染函数 `sub_1000073E8` 在解锁后把地图中间替换为：

```text
 VM UNLOCKED
```

### 3. flag 生成前的状态校验

`updateWithFrame:` 发现解锁标志第一次置位后，先由 `sub_100007E04` 导出一个 56 字节状态快照，再调用 `sub_100005C7C`：

```c
sub_100007E04(game_state);
if (sub_100005C7C(game_state, output, 128))
    flagLabel = output;
else
    flagLabel = "error";
```

状态结构可以按访问偏移还原为：

```c
struct GameState {
    uint64_t seed;          // +0x00
    uint64_t rng_state_1;   // +0x08
    uint64_t rng_state_2;   // +0x10
    uint64_t elapsed_ticks; // +0x18
    uint32_t moves;         // +0x20
    uint32_t score;         // +0x24
    uint32_t beans;         // +0x28
    uint32_t fast_moves;    // +0x2c，单步不超过 11 ms
    uint32_t min_step_ms;   // +0x30
    uint32_t max_step_ms;   // +0x34
};
```

`sub_100006668` 会验证：

1. `score >= 10000`；
2. `score == 10 * beans`；
3. `moves >= beans`；
4. 随机种子、两个随机状态和总耗时均非零；
5. `min_step_ms != 0` 且 `max_step_ms >= min_step_ms`；
6. 平均步间时间至少约 8 ms；
7. 小于等于 11 ms 的快速按键数不超过总步数的三分之一；
8. 随机状态已经前进，不能仍等于种子直接派生出的初始值。

因此直接把分数变量改成 10000 仍然过不了检查，自动以极高频率点击也会被 `fast_moves` 条件拒绝。

### 4. 反分析逻辑

应用启动时会创建后台线程，每 3 秒执行一次检查。相关代码主要完成两件事：

- 检查关键函数开头是否被替换为 ARM64 跳板，如直接 `B/BL/BR`、`ADRP + LDR + BR` 等常见 inline hook 形式；
- 通过私有 XPC 接口枚举 PID，并在进程的 `label` 和 `path` 中查找 `frida`、`objection`、`substrate`、`jailbreakd` 等字符串。

一旦命中，程序会设置全局污染标志，后续 flag 生成失败。这个设计会显著增加真机动态调试的成本，但所有 actor 描述、加密参数和密文都在 Mach-O 中，因此没有必要和反调试逻辑正面对抗，静态模拟状态机即可。

### 5. 还原 Mach actor 状态机

`sub_100005C7C` 的核心初始值为：

```c
uint64_t key = 0x13895CA3BAFED00D;
uint32_t actor_index = 39;
```

72 个 actor 从虚拟地址 `0x10000A800` 开始，每项 24 字节：

```c
struct Actor {
    uint16_t token;
    uint8_t  argc;
    uint8_t  tag;
    uint32_t payload_index;
    uint64_t nonce;
    uint64_t salt;
};
```

actor 的参数区从 `0x10000AEC0` 开始，以 64 位小端整数排列。

`sub_100006C48` 建立 4 个 Mach port，并为每个端口启动 `sub_100006D2C` worker。主线程通过 `sub_100006384` 发送 actor 索引、token、参数描述、当前 key 和完整性字段。不同 token 被路由到不同处理器：

| token | 处理逻辑 |
|---:|---|
| `0x71C3` | `sub_100005F18`，最多解码 3 个参数 |
| `0xC4A7` | `sub_100006100`，最多解码 2 个参数 |
| `0x39E1` | 合法终止 actor |
| `0xA913` | 故意设置的失败终止 actor |
| 其他值 | 通用 worker 分支 |

程序反复使用一个 SplitMix64 风格的混合函数。反编译中的 `x - 0x61C8864680B583EB` 等价于模 $2^{64}$ 加 `0x9E3779B97F4A7C15`：

```python
MASK64 = (1 << 64) - 1

def mix(x):
    x = (x + 0x9E3779B97F4A7C15) & MASK64
    x = ((x ^ (x >> 30)) * 0xBF58476D1CE4E5B9) & MASK64
    x = ((x ^ (x >> 27)) * 0x94D049BB133111EB) & MASK64
    return x ^ (x >> 31)
```

对 actor 的第 `j` 个参数，解码公式是：

```python
base = actor.salt ^ key ^ (actor.token << 32) ^ actor_index
arg[j] = encoded[j] ^ mix(base ^ (j * 0x517CC1B727220A95))
```

Mach 消息里还有随机会话值、`nonce` 和校验和。它们用于确认请求和响应没有被简单伪造，但客户端只取下一 actor 索引的低 32 位；对应掩码会在请求端和 worker 响应端互相抵消。离线求最终 key 时可以把状态转移化简为：

```python
if token == 0x71C3:
    # 不足三个参数时以 0 补齐
    next_actor = arg2 & 0xffffffff
    response_key = mix(
        key ^ salt ^ arg0 ^ arg1 ^ (arg2 & 0xffffffff)
    )
elif token == 0xC4A7:
    # 不足两个参数时以 0 补齐
    next_actor = arg1 & 0xffffffff
    response_key = mix(
        key ^ salt ^ arg0 ^ (arg1 & 0xffffffff)
    )
else:
    next_actor = (actor_index + (salt & 7) + 1) % 72
    response_key = mix(key ^ salt ^ next_actor)

key = mix(response_key ^ key)
actor_index = next_actor
```

合法终止 actor `0x39E1` 还要求：

```python
arg0 == mix(actor.salt ^ key)
```

通过验证时不再更新 key，当前值就是后续 RC4 的 64 位密钥。

一个容易误判的点是：`0x13895CA3BAFED00D` 紧挨着最终解密调用，看起来很像固定 RC4 密钥。直接用它解密只会得到非 ASCII 数据，因为该值只是 actor 链初始状态；每经过一个 actor，`key` 都会再次更新。

离线模拟得到的 actor 路径为：

```text
39 -> 5 -> 22 -> 55 -> 40 -> 31 -> 50 -> 24 -> 49 -> 4
   -> 27 -> 44 -> 47 -> 29 -> 16 -> 10 -> 2 -> 9 -> 6 -> 18
   -> 45 -> 23 -> 58 -> 11 -> 61 -> 26 -> 64 -> 52 -> 14
```

第 29 个节点，即 actor 14，是通过验证的终止 actor。最终 key 为：

```text
0x5ead5b71a04140a7
```

### 6. RC4 解密

`sub_100006A1C` 从 `0x10000A700` 复制 `0, 1, ..., 255`，执行标准 RC4 KSA 和 PRGA。64 位整数 key 在 ARM64 小端环境下按以下 8 字节参与 KSA：

```text
a7 40 41 a0 71 5b ad 5e
```

40 字节密文位于 `0x10000B3A0`：

```text
3b 9e 14 5d 9d c7 22 95 90 77
88 ec ee 4a b0 cf ec df eb 5d
85 ab eb 91 60 81 e6 98 a7 ae
86 65 b1 3d e3 d3 95 9e a5 56
```

求解脚本可以直接从 `pacman.ipa` 读取 Mach-O，无需预先解包，也不依赖第三方库。处理步骤为：

1. 解析 72 项 actor 表；
2. 解码 actor 参数并模拟状态转移；
3. 验证终止 actor；
4. 取最终 key 的小端字节序执行 RC4；
5. 检查结果格式并进行 RC4 往返校验。

运行方式：

```bash
python solve.py pacman.ipa
```

关键输出：

```text
actor steps:         29
actor path:          39 -> 5 -> 22 -> 55 -> 40 -> 31 -> 50 -> 24 -> 49 -> 4 -> 27 -> 44 -> 47 -> 29 -> 16 -> 10 -> 2 -> 9 -> 6 -> 18 -> 45 -> 23 -> 58 -> 11 -> 61 -> 26 -> 64 -> 52 -> 14
final key (u64):     0x5ead5b71a04140a7
key (little-endian): a74041a0715bad5e
ciphertext length:   40
plaintext:           d3ctf{GoOdjob!!!Y0u_@re_be5t_P4c-Man!!!}
```

最终 flag：

```text
d3ctf{GoOdjob!!!Y0u_@re_be5t_P4c-Man!!!}
```

## 方法总结

- 核心技巧：从 Objective-C UI 方法反向定位真正的 flag 调用链，再把跨 Mach port 的 actor 协议还原为离线状态转移，最后恢复 RC4 key。
- 识别信号：游戏界面、极高分数门槛、`VM UNLOCKED`、Mach API、静态 actor 表和大量反调试字符串同时出现时，应优先判断游戏是否只是状态收集外壳。
- 复用要点：不要把紧邻解密调用的常量直接当最终密钥；先检查它是否在循环中被反复更新。面对复杂 IPC 时，也不一定要完整模拟内核通信，只要找出请求字段、响应字段和客户端最终使用的位宽，往往可以消掉传输层完整性噪声，留下确定性的状态机。
- 动态调试并非本题最短路线。程序专门检查 inline hook、Frida 和越狱工具，而静态 actor 表、密文和算法已经足够恢复 flag，离线模拟更稳定、也更容易验证。
