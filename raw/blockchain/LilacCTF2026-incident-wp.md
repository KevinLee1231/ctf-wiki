# Incident WP

## 题目简述

这是一道 Solana runtime 题。题目表面上是 marginfi 相关链上交互，实际预期漏洞点在 Solana direct mapping 与 CPI 边界状态同步之间，相关修复可参考 Anza Agave PR 6709。服务会加载 marginfi 程序并要求构造异常账户状态，因此需要利用 runtime bug 创建任意 owner 的账户并控制其内部数据。

## 解题过程

Solana direct mapping 模式下，账户数据直接映射到 rbpf VM 的内存区域。为了性能，写内存时不会每次重新判断“账户 owner 是否仍是当前程序”，而是依赖账户进入指令时创建的 `MemoryRegion.writable`。这个 writable 位在序列化账户输入时确定：账户必须在 `AccountMeta` 中可写，且进入指令时 owner 是当前程序。

漏洞链如下：

1. 进入 Program A 时，账户 X 属于 Program A，并且以 writable 传入，因此 X 对应的 `MemoryRegion` 是 writable。
2. Program A 在执行过程中把 X 的 owner 改成 Program B。
3. Program A 发起 CPI，并把 X 作为 readonly account 传给被调用程序。
4. CPI 入口的 `update_callee_account` 会把 caller VM 中修改后的 owner 写回 transaction-level `AccountSharedData`，因此全局状态里 X 已经属于 Program B。
5. CPI 返回时，runtime 只在账户以 writable 传入 CPI 或触发特定 `update_caller` 条件时更新 caller VM 状态。因为这里把 X 作为 readonly 传入，且没有数据长度变化，caller VM 中旧的 `MemoryRegion` 没有刷新。
6. Program A 回到 CPI 之后继续写 X 的数据。写入检查仍看到原来的 writable `MemoryRegion`，于是写入成功。
7. Program A 结束时，反序列化阶段看到 transaction-level owner 已经是 Program B，不会再触发“外部账户数据被修改”的检查。

结果是：Program A 可以在逻辑上把账户 owner 改给任意 Program B 后，继续写这个已经不属于自己的账户数据。这样可以构造一个任意 owner 的账户，并控制其内部数据。

题目选择 marginfi 作为利用目标，是因为 marginfi 的部分账户结构允许非 PDA 账户存在。如果目标协议强制 PDA 校验，即使能伪造 owner 和数据，也不一定能通过业务层地址校验。

## 方法总结

本题的关键是区分三份状态：caller VM 中的账户输入、transaction-level `AccountSharedData`、以及 CPI callee 看到的账户状态。owner 变化会在 CPI 入口同步到 transaction-level 状态，但 readonly CPI 返回后没有刷新 caller VM 的 writable `MemoryRegion`，造成权限过期。审计 Solana runtime 类题目时，要重点看 CPI 边界是否同时同步数据、owner、lamports、长度和内存映射权限。
