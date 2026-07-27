---
type: technique
tags: [blockchain, evm, delegatecall, proxy, storage-layout, technique]
skills: [ctf-blockchain]
raw:
  - ../raw/blockchain/0xGame2023-week3-0xcallsino-wp.md
  - ../raw/blockchain/SekaiCTF2026-pp-farming-2-wp.md
updated: 2026-07-27
---

# Delegatecall Storage Layout Corruption

## 适用场景

EVM 合约通过 proxy、plugin、library 或用户可选实现执行 `delegatecall`，目标状态却存放在调用方。攻击者不一定需要实现代码本身有传统漏洞，只要被调用代码能按不兼容布局写入关键 slot，就可能覆盖 implementation、admin、锁或业务状态。

## 识别信号

- 合约把地址存入状态变量，随后对该地址执行 `delegatecall`。
- helper/implementation 可由用户注册、替换或间接影响。
- 调用方与实现合约变量顺序、继承层次或类型宽度不同。
- 重入锁、owner 或 implementation 在某次委托调用后异常改变。

## 最小证据

- 列出调用方和每个实现的 slot 编号、offset、类型及继承来源。
- 确认委托代码运行时的 `address(this)`、`msg.sender` 和 storage context。
- 找到至少一个攻击者可触发的写操作及其实际落入的调用方 slot。
- 证明被覆盖状态能改变后续调用目标、授权或资金流。

## 解法骨架

1. 从源码、编译布局或 `eth_getStorageAt` 画出逐 slot 对照表。
2. 追踪所有 `delegatecall` 入口及实现地址的写入路径。
3. 先用合法实现或布局碰撞覆盖关键地址/锁，再让后续调用进入攻击者实现。
4. 恶意实现按调用方布局写目标 slot，不按自身变量名推断效果。
5. 每一步读取链上 storage，确认覆盖顺序和交易原子性。

## 关键变体

- 实现地址本身位于 slot 0 时，普通初始化函数就可能成为第一次劫持原语。
- EIP-1967 等非顺序 slot 能降低直接碰撞，但任意实现和未授权升级仍是独立风险。
- 锁变量被委托代码覆盖时，问题不是锁算法，而是锁状态不再由可信代码独占。
- packed storage 需要同时计算 slot 与字节 offset，整槽写可能破坏相邻字段。

## 常见陷阱

- 按 Solidity 变量名理解写入位置，没有比较实际 storage layout。
- 只审计 proxy 自身代码，忽略所有可达 implementation/helper 的写函数。
- 看到重入结果就只查外部调用顺序，漏掉锁 slot 被直接覆盖。
- 测试时只看事件或 getter，没有读取关键 slot 验证真实状态。

## 关联技巧

- [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md)
- [bridge-message-field-binding.md](bridge-message-field-binding.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [0xGame2023-week3-0xcallsino-wp.md](../raw/blockchain/0xGame2023-week3-0xcallsino-wp.md)
- [SekaiCTF2026-pp-farming-2-wp.md](../raw/blockchain/SekaiCTF2026-pp-farming-2-wp.md)
