# DownUnderCTF 2022 Private Log Writeup

## 题目简述

`PrivateLog` 作为 `TransparentUpgradeableProxy` 的逻辑合约运行，代理每分钟会发送一笔创建日志的交易。`createLogEntry` 和 `updateLogEntry` 都要求提交旧密码，并把 `secretHash` 换成调用者指定的新哈希。日志更新为了节省 gas 使用了内联汇编：

```solidity
assembly {
    let length := mload(logEntry)
    mstore(0x00, logEntries.slot)
    sstore(
        add(keccak256(0x00, 0x20), logIndex),
        or(mload(add(logEntry, 0x20)), mul(length, 2))
    )
}
```

代码没有检查 `logIndex < logEntries.length`。目标是先夺取滚动密码，再把越界数组写转化为任意 storage 写，最终改写代理的实现地址并转走余额。

## 解题过程

### 抢占密码状态

监听 pending 交易可直接从 calldata 解出定时任务提交的明文 `password`。题目区块时间较长，使用更高 gas price 抢先调用 `createLogEntry`，同时把 `newHash` 设置为自选密码的哈希。攻击交易先上链后，原交易会因旧密码失效而回滚，`secretHash` 就由玩家控制。

### 把数组索引映射到代理槽

`logEntries` 位于 slot 2，动态数组元素从

$$
B=\operatorname{keccak256}(\operatorname{pad}_{32}(2))
$$

开始存储。EIP-1967 规定代理实现地址位于

$$
S=\operatorname{keccak256}(\text{"eip1967.proxy.implementation"})-1.
$$

由于汇编算术按 $2^{256}$ 回绕，令

$$
\text{logIndex}\equiv S-B\pmod{2^{256}},
$$

`sstore(B + logIndex, value)` 就会写到实现槽。

短字符串的 31 字节数据左对齐，最后一字节会被 `31 * 2 = 0x3e` 占用。先按部署者地址和 nonce 预测合约地址，不断推进 nonce，直到待部署攻击合约地址以 `0x3e` 结尾。随后构造 31 字节日志值：12 个零字节、攻击地址前 19 字节和一个零字节；汇编与 `0x3e` 做 OR 后，恰好形成右对齐的完整攻击合约地址。

攻击合约复用逻辑合约的 `init(bytes32)` 选择器，但函数体只转账：

```solidity
contract AttackPrivateLog {
    function init(bytes32) public payable {
        payable(msg.sender).transfer(address(this).balance);
    }
}
```

发送手工 ABI 编码的 `updateLogEntry` 覆盖实现槽，再通过代理调用 `init`。代理执行 `delegatecall`，因此 `address(this).balance` 指向代理余额，资金被转给玩家。平台最终返回：

```text
DUCTF{first_i_steal_ur_tx_then_I_steal_ur_proxy_then_i_steal_ur_funds}
```

## 方法总结

本题把三种链上机制串成完整利用链：mempool 抢跑夺取滚动凭据、动态数组越界索引产生模 $2^{256}$ 的任意槽写、EIP-1967 实现槽被覆盖后利用 `delegatecall` 接管代理。审计汇编 storage 操作时必须补回 Solidity 本应执行的边界检查，并检查写入 primitive 是否能够触达代理管理槽；短字符串编码的长度字节也可能限制可写值，需要通过可预测合约地址补齐约束。
