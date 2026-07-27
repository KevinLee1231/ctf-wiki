---
type: technique
tags: [pwn, protocol, parser, length, state-machine, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/ACTF2022-master-of-dns-wp.md
  - ../raw/pwn/SekaiCTF2026-ppp-wp.md
  - ../raw/pwn/SUCTF2026-evbufferWP.md
updated: 2026-07-27
---

# Protocol Length, State and Parser Corruption

## 适用场景

二进制协议、压缩格式或事件驱动网络程序分别维护线缆长度、解压后长度、分片长度、总包长度和会话对象状态。单个字段可能各自通过检查，但字段间关系、整数转换或跨协议共享状态仍会导致栈/堆溢出、对象覆盖或验证绕过。

## 识别信号

- `this_length`、`entire_length`、record length、decompressed length 等字段参与分配和接收。
- 压缩指针或编码层能让短报文展开成更长逻辑数据。
- 长度从大整数截断到 32 位，或先减后比较，存在下溢。
- TCP/UDP 等多个 callback 共享相邻全局 session、buffer 或第三方库对象。
- 验证函数只消费文本前缀，二进制尾部仍进入后续处理。

## 最小证据

- 区分线上报文字节数、解析后逻辑长度、分配长度和最终复制长度。
- 写出所有长度字段之间应满足的不变量，并定位检查发生在转换/减法前还是后。
- 还原跨 callback 的状态布局和事件时序，确认哪个包写、哪个事件触发使用。
- 用最小畸形包证明越界或状态覆盖，不先依赖完整 ROP/heap 链。

## 解法骨架

1. 按协议规范写最小序列化器/解析器，逐字段记录 offset、宽度和端序。
2. 建立长度关系，例如 `header <= fragment <= total <= allocation`，检查每次类型转换。
3. 构造只破坏一个关系的报文，观察分配、接收、解压和复制的实际参数。
4. 将越界落点映射到栈帧、tcache metadata、相邻 session 或库对象的最小必要字段。
5. 结合运行环境保护选择最短利用链，并按真实网络状态机稳定触发。

## 关键变体

- DNS 压缩指针可让编码长度很短而逻辑名称超出固定缓冲区。
- 分片长度与总长度分别合法但关系反转时，减法和窄化可产生超大接收长度。
- 事件驱动程序中，一个协议负责泄漏，另一个协议负责覆盖和触发。
- 第三方库对象伪造应只填充真实调用链读取的字段，并绑定附件版本。

## 常见陷阱

- 把抓包长度直接当作解压/复制长度。
- 逐字段做上界检查，却没有验证字段之间的偏序关系。
- 机械加入 libc 泄漏，未先确认 ASLR、libc 和 safe-linking 是否已固定或关闭。
- 只审计单个 callback，漏掉共享全局对象和异步事件顺序。
- 照搬其它版本库结构偏移，没有在附件二进制中验证。

## 关联技巧

- [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)
- [overflow-basics.md](overflow-basics.md)
- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [pwn-tooling.md](pwn-tooling.md)

## 原始资料

- [ACTF2022-master-of-dns-wp.md](../raw/pwn/ACTF2022-master-of-dns-wp.md)
- [SekaiCTF2026-ppp-wp.md](../raw/pwn/SekaiCTF2026-ppp-wp.md)
- [SUCTF2026-evbufferWP.md](../raw/pwn/SUCTF2026-evbufferWP.md)
