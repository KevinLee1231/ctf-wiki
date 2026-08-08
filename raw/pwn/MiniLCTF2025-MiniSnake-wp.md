# MiniSnake

## 题目简述

程序是一个 ncurses 贪吃蛇游戏。地图 `uint8_t map[16][16]` 位于 `game()` 栈帧中，蛇身在每帧绘制前按坐标写入地图。事件线程同时处理“吃到点数”和“撞墙”，但显示 `GOT POINT!` 时会睡眠 0.4 秒；主游戏线程在此期间仍持续移动。若吃到点后立刻撞墙，事件线程可能来不及终止游戏，蛇头越过边界后形成栈上越界写。Numeric skin 又让三个初始蛇身字节由种子确定，因此可把返回地址低三字节改为 `admin_env`。

当前 ELF 为 amd64、动态链接、未 strip；保护为 Partial RELRO、栈 canary、NX、无 PIE。当前附件的 `admin_env=0x40155e`，所以只需改写 saved RIP 低三字节即可保留高位并跳入后门。

## 解题过程

### 事件线程为何会错过撞墙

吃到食物后，主线程设置 `ate_point_idx`。事件线程处理该事件时先增长蛇身，然后显示提示并休眠：

```c
if (ate_point_idx != -1) {
    snake->body[snake->length++] = snake->points[ate_point_idx];
    ...
    thrd_sleep(&(struct timespec){.tv_nsec = TICK_TIME * 2}, NULL);
}

if (hit_wall) {
    running = false;
}
```

`hit_wall` 的处理位于睡眠之后。主线程每 0.2 秒继续执行 `snake_move → emit_event → set_map`，因此在 0.4 秒窗口内至少还能推进一帧。

边界判断只接受恰好位于 `-1`、`WIDTH` 或 `HEIGHT` 的坐标：

```c
if (x == -1 || y == -1 || x == WIDTH || y == HEIGHT)
    hit_wall = true;
else
    hit_wall = false;
```

撞墙的第一帧，`set_map` 因 `hit_wall=true` 从 `body[1]` 开始，暂时跳过越界蛇头。下一帧蛇头已从 `WIDTH` 移到 `WIDTH+1`，不再满足“恰好等于边界”的判断，`hit_wall` 被清回 false；`set_map` 又从 `body[0]` 开始，执行未经边界检查的：

```c
map[snake->body[i].y][snake->body[i].x] = snake->body[i].value;
```

从此每移动一格都能向 `map` 后方的栈内存写一个 Numeric skin 字节。

### 定位 saved RIP 而不破坏 canary

返回地址位于地图之后，但中间还有 stack canary。官方方法不是盲猜偏移，而是在本地副本中临时扩大 `draw` 的行数和窗口高度，把地图外的栈字节按十六进制显示出来。观察扩展后的两行可识别出 canary、saved RBP 与形如 `0x40xxxx` 的 saved RIP，从而记录从边界到返回地址的坐标路线。

这个 patch 只用于定位，不能直接用于远程；最终路线必须让蛇身字节落在 saved RIP 上，同时避免改坏其上的 canary。由于地图按 `map[y][x]` 线性存储，越界地址为：

$$
\&map[0][0]+16y+x
$$

因此调试输出中的目标字节偏移可以直接换算为越界后的移动步数。

### 选择能编码后门地址的种子

Numeric skin 的前三个蛇身值依次为：

```c
random() % 254 + 1
```

官方 WP 的旧构建中，种子 `16183281` 产生 `40 16 4d`，沿官方路径由蛇尾到蛇头反向落入内存后形成低三字节 `4d 16 40`，即旧地址 `0x40164d`。

当前仓库二进制已经发生地址漂移：`admin_env` 实际为 `0x40155e`。直接复用旧种子只会写向错误地址。按同一字节顺序需要 PRNG 产生 `40 15 5e`；对 glibc `srandom/random` 的有界搜索得到候选种子：

```text
4899864 → 0x40, 0x15, 0x5e
```

搜索逻辑如下：

```c
for (unsigned seed = 0; seed < INT_MAX; ++seed) {
    srandom(seed);
    if (random() % 254 + 1 == 0x40 &&
        random() % 254 + 1 == 0x15 &&
        random() % 254 + 1 == 0x5e) {
        printf("%u\n", seed);
        break;
    }
}
```

该候选已验证 PRNG 输出，但当前仓库没有保存方向按键脚本，且官方路线坐标来自旧构建，仍需针对当前二进制重新定位 saved RIP 并实机确认路线，不能把 `4899864` 单独视为完整远程 exploit。

### 劫持返回并进入维护模式

实际利用顺序为：选择 Fixed seed、输入与当前后门地址匹配的种子、选择 Numeric skin；按预先记录的路线先吃到一个点，在事件线程显示 `GOT POINT!` 的睡眠窗口内立即撞墙；继续控制蛇越界移动，使三个蛇身字节覆盖 saved RIP；按 `Q` 令 `game()` 返回。返回地址进入 `admin_env` 后选择 3，程序执行：

```c
system("/bin/sh");
```

官方旧构建验证得到：

```text
miniLCTF{secret_destination_behind_walls}
```

## 方法总结

- 核心技巧：利用事件线程中的阻塞窗口错过撞墙状态，把游戏坐标推进为栈上 OOB 写，再用可控 PRNG 字节部分覆盖 saved RIP。
- 识别信号：边界检测只检查“恰好等于边界”、处理线程会 sleep、主线程仍持续更新共享状态，是典型的竞态穿界信号。
- 复用要点：canary 存在时应借可视化或本地 patch 精确定位，不能线性扫写；种子、函数地址和路线都受构建版本影响，旧 WP 的种子必须和当前二进制符号重新核对。
