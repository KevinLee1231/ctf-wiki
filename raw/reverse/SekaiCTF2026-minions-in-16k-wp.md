# Minions in 16k

## 题目简述

这是一个开放式 King-of-the-Hill 游戏题。选手只拿到约 62 MB、未剥离符号的 Bevy/Rust FPS 客户端，服务端在比赛期间保持闭源；游戏内部渲染分辨率为 $160\times100=16000$ 像素。前 10 场是每场 1 分的校准局，之后获得 Elo，平台再把 Elo 映射为题目分数。它没有固定静态 flag，真正交付物是能持续参加约 5 分钟对局并取得正收益的 bot。

## 解题过程

### 恢复比赛版本协议

先对 Linux 客户端运行 `nm`、`strings` 并结合反汇编定位 `client::net`、`client::autopilot`、`sim::world`、`sim::weapon` 等模块。客户端不带 `--remote` 时可以离线与内置 AI 对战，适合在消耗线上实例前验证控制逻辑。

网络使用 QUIC 和 bincode 1.3 默认编码：固定宽度整数为小端，枚举 tag 是 `u32`，字符串和序列长度是 `u64`。每个出站 UDP 数据报前还要添加 16 字节 instance-id，供前置路由器分流；实现自定义 `quinn::AsyncUdpSocket` 时在发送路径加前缀、接收路径按实际服务行为剥离。客户端原本校验服务端 DER 证书的 SHA-256，bot 只连接题目实例时也可以安装 accept-any verifier。

双向 QUIC stream 的帧格式为：

```text
[u32 little-endian body_len][bincode body]
body = [u32 little-endian variant_tag][fields...]
```

关键客户端消息：

```text
Join  { protocol_version:u16 = 11, name:String, key:Option<String>, skin:u8 }
Input { tick:u64, buttons:u16, yaw:f32, pitch:f32 }
Quit
```

比赛期 bot 源码把 `Join` 的首字段命名为 `channels` 并发送 11；字段名本身不会进入 bincode，赛后公开源码将同一语义明确命名为 `protocol_version`。赛后源码的其它协议结构还在继续演进，因此精确 wire layout 应以比赛发放二进制和实际抓包为准，不能直接套用当前源码树。

关键世界更新为：

```text
[tag=7][tick:u64][count:u64][count * 30-byte Snapshot]

Snapshot {
    victim: u32,
    id: u8,
    pos: [i32; 3],
    dir: [i32; 3],
    active: u8
}
```

`count` 至多为 8。位置和方向是有符号 16.16 定点数，要除以 65536；早期若误判为 `i16` 和 9 字节 stride，会得到数百个实体及饱和坐标。收到 tag 2 的 `Welcome.player_id` 后，用 `Snapshot.id` 区分自身与敌人。

### 构造战斗 bot

Bevy 前方为 $-Z$，`client::autopilot` 中的权威瞄准公式为：

$$
\text{yaw}=\operatorname{atan2}(-\Delta x,-\Delta z),
$$

$$
\text{pitch}=\operatorname{atan2}\left(\Delta y,\sqrt{\Delta x^2+\Delta z^2}\right).
$$

无头 bot 完成如下循环：

1. 建立带 instance-id 前缀的 QUIC 连接，打开双向 stream 并发送 `Join`；
2. 从 `Welcome` 保存自身 id，每个 64 Hz tick 解码最新 `PlayerUpdate`；
3. 过滤原点和越界坐标，并用自身 id/位置排除自己；不能仅凭 `active=false` 判死，再按距离、生命值等指标选择敌人；
4. 用相邻快照估计速度，只对快照陈旧造成的位移做提前量；武器是 hitscan，不需要加入弹道飞行时间；
5. 计算 yaw/pitch，按需要前进、横移、跳跃并发送 `Input`。

最关键的逆向发现是 `FIRE` 为上升沿触发。若每一 tick 都保持 FIRE 位，角色每条命实际上只射出第一发。正确方式是奇偶 tick 交替：

```rust
let buttons = if input_tick % 2 == 0 { FIRE } else { 0 };
```

这样每两 tick 产生一次 $0\rightarrow1$，比赛版本可达到约 32 次触发/秒。保持出生点掩体、追逐进入有效距离、横移躲避并持续脉冲开火后，bot 才从“能连接”变成有正 Elo 期望的对手。完整的协议恢复、对局经验和 bot 源码见这份[固定提交的实战题解](https://github.com/ZeroSolves/ctf-writeups/blob/dda99e43af24685bd35457d6be0a813fadee267a/sekai-2026/minions-in-16k/README.md)；影响复现的协议字段、瞄准公式和射击节奏已在正文展开。

## 方法总结

本题的交付不是固定 flag，而是可持续参赛的协议实现和战斗 bot。连上服务器只完成了一半：评分是零和 Elo，净负收益 bot 持续运行会主动掉分，达到高水位后停机也是合理策略。应先在离线模式验证协议和控制逻辑，再以本题独立 Elo/净胜率监控线上效果。由于主要障碍是从客户端恢复网络布局、定点数与输入语义，本篇从 `_unclassified` 调整到 `reverse`。
