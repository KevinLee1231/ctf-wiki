# level5

## 题目简述

程序把商店、背包与战斗封装成一个小游戏。全局状态 `g` 依次保存金币、属性、`banner_len`、`stage`、八格背包和 `bag_cnt`。药水升级由分离线程执行，中间存在两秒休眠且没有锁；背包计数因此可以被竞争到负数。后续购买会按 `g.bag[g.bag_cnt]` 写入，形成全局结构体负索引写。

目标为 amd64 ELF，保护为 Partial RELRO、无栈 canary、NX 开启、无 PIE。利用链是：竞态双卖使 `bag_cnt=-1` → 负索引把 `stage` 写成 10 → `quit_game` 精确覆盖 saved `rbp` → 借 `game` 的名字输入实现定址写 → 放大 `banner_len` 泄露 libc → 再放大 `stage` 触发完整栈溢出并 ret2libc。

## 解题过程

### 竞争两次出售同一瓶药水

先击败第一关史莱姆取得 50 金币，再购买一瓶 Small Potion。升级线程先确认槽位中是小药水，然后休眠两秒：

```c
static void *brew(void *arg) {
    int i = (int)(intptr_t)arg;
    if (i < 0 || i >= BAG) return NULL;
    if (g.bag[i] != SMALL) return NULL;
    sleep(2);
    g.bag[i] = 0;
    g.bag[i] = BIG;
    return NULL;
}
```

休眠期间出售槽位 0，`bag_cnt` 从 1 降到 0；线程醒来后仍把槽位 0 写成 Big Potion。再次出售槽位 0 时，`sell_item` 的检查是 `i > g.bag_cnt` 而不是 `i >= g.bag_cnt`，所以 `i=0, bag_cnt=0` 仍能通过，第二次出售后 `bag_cnt=-1`。

### 用 `bag[-1]` 改写 `stage`

`g` 的关键偏移如下：

```text
g+0x18  banner_len
g+0x1c  stage
g+0x20  bag[0]
g+0x40  bag_cnt
```

因此 `bag[-1]` 正好别名 `g.stage`。选择商店第 8 项 Blue Gem，其内部 ID 为 `GEM=10`，购买语句会执行：

```c
g.bag[g.bag_cnt] = it->id;  // bag[-1] = 10
g.bag_cnt++;                // -1 -> 0
```

两次出售还会退回金币，所以在第一关获得的 50 金币足以完成小药水、两次出售与蓝宝石购买。

### 精确控制 `game` 的 `rbp`

`quit_game` 的反汇编确认局部缓冲区为 `rbp-0x40`，读取长度为：

$$
4\times(\text{stage}+8)
$$

当 `stage=10` 时长度恰为 `0x48`：前 `0x40` 字节填满缓冲区，后 8 字节只覆盖 saved `rbp`，还不会碰 saved RIP。函数执行 `leave; ret` 后返回 `game`，但 `game` 的 `rbp` 已变成攻击者指定值。

`game` 的名字缓冲区使用 `[rbp-0x50]`，并执行 `read(0, rbp-0x50, 0x40)`。于是把伪造的 saved `rbp` 设置为：

$$
\text{target}+0x50
$$

下一次选择 `name` 就会向 `target` 写入 64 字节。二进制中 `g=0x4050a0`，若令 `target=g+0x18=0x4050b8`，即可一次控制 `banner_len`、`stage`、`bag[]`、`bag_cnt` 和 `own`。对应伪造 `rbp` 为 `0x405108`。

### 放大读取并完成 ret2libc

先通过定址写把 `banner_len` 改为大于 `board[0x40]` 的值。`battle` 在检查关卡是否已经全部通关之前就执行：

```c
char board[0x40] = "challenger rank";
write(1, board, g.banner_len);
```

即使 `stage=10`，仍会先从 `board` 向后泄露栈内容。根据实际输出定位栈中的 libc 返回地址并减去对应符号偏移，得到 libc 基址：

$$
\text{libc base}=\text{leaked return address}-\text{return-site offset}
$$

随后再次使用 `name` 定址写，把 `stage` 改为足够大的正数。再选择退出时，`4(stage+8)` 将明显超过 `0x48`，从覆盖 saved `rbp` 扩大为可控 saved RIP。目标无 canary 且无 PIE，按泄露出的 libc 基址布置 `ret` 对齐、`pop rdi; ret`、`"/bin/sh"` 和 `system` 即可完成 ret2libc。

可复现的交互顺序为：

```text
战胜第 1 关
商店买 Small Potion
upgrade(0)
在 2 秒内 sell(0)
等待 [brew] done
再次 sell(0)              # bag_cnt = -1
购买 Blue Gem             # stage = 10
exit，发送 0x40 填充 + p64(0x405108)
name，写入放大的 banner_len 与受控 stage
battle，解析越界输出中的 libc 地址
name，再次写大 stage
exit，发送 ret2libc payload
```

原资料没有提供最终 exploit、libc 文件或 flag；因此 libc 返回点偏移、`system` 偏移和竞态交互延迟必须在实际 Ubuntu 24.04 运行环境中匹配，本文不编造这些环境相关常量。

## 方法总结

- 核心技巧：用无锁线程制造对象“已售出又被恢复”的状态，双卖把计数降为负数，再把全局负索引写逐步放大为 `rbp` 控制、定址写、地址泄露和完整栈溢出。
- 识别信号：后台线程在检查对象后休眠，主线程却能同时删除对象；同时计数器被直接用于数组写下标，这是典型的竞态到内存破坏链。
- 复用要点：不要急于一次覆盖 RIP。精确写满局部缓冲区再只覆盖 saved `rbp`，可借调用者的相对寻址输入构造稳定任意写；所有长度都应由 `4(stage+8)` 明确推导。
