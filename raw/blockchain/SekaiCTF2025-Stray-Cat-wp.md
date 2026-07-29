# Stray Cat

## 题目简述

目标合约 `Cat` 中的 `gibHeadPats()` 会先回滚，因此正常交易不可能成功触发 `Purr()` 事件。检查器却不直接查询合约状态，而是调用调试接口读取区块的原始交易回执，自己用 Python RLP 库解析，并检查其中是否存在：

- 状态为成功的交易回执；
- 地址等于 `Cat` 合约；
- 首个 topic 等于 `keccak256("Purr()")`；
- data 为空的日志。

因此主障碍是原始回执在不同解析边界下的歧义，而不是绕过 Solidity 中的 `revert`。

## 解题过程

### 1. 确认正常路径不可行

目标函数的逻辑等价于：

```solidity
function gibHeadPats() external {
    revert();
    emit Purr();
}
```

即使手工构造 calldata，EVM 也不会提交回滚后的日志。必须让检查器“解析出”一个并未由 `Cat` 合约真实发出的日志。

### 2. 对齐检查器的原始回执解析

检查器通过 `debug_getRawReceipts` 获取字节串，去除 typed receipt 的类型字节后交给 RLP 解码器。官方解法利用回执编码与该解析流程的边界差异，将一个伪造的日志 RLP 结构嵌入攻击合约真实事件的大块 data 中。

嵌入的核心对象为：

```python
fake_log = [
    bytes.fromhex(cat_address[2:]),
    [keccak(b"Purr()")],
    b"",
]
embedded = rlp.encode([fake_log])
```

这里的地址必须按当前实例重新生成，不能复用固定部署地址。

### 3. 构造受控事件数据

官方 `gen.py` 生成约 63 KB 的事件数据。其结构可概括为：

```text
大段 A 填充
RLP 长度控制字节
伪造的 Cat/Purr 日志 RLP
另一组 RLP 长度控制字节
大段 B 填充
```

官方载荷在 A 区的特定偏移写入 `b9 01 00`，在后续位置嵌入伪日志，再写入 `b9 ff ff`。这些长度字段让真实 EVM 回执仍能成立，却让检查器使用的解码路径把 data 内的字节重新解释为外层日志列表的一部分。

攻击合约先发出携带这段 data 的自身事件：

```solidity
event SolveLog(bytes data);

function solve(bytes calldata data) external {
    emit SolveLog(data);
    assembly {
        for { } gt(gas(), 100) { } { }
    }
}
```

末尾主动消耗剩余 gas，是为了让最终回执的编码长度与官方构造所依赖的边界稳定对齐。

### 4. 提交精确区块

官方流程为：

1. 部署解题合约；
2. 根据当前 `Cat` 地址运行生成器；
3. 进行两次预热调用；
4. 用指定 gas 发送 legacy 交易调用 `solve`；
5. 将包含该交易的区块号提交给检查器。

链上看到的是攻击合约发出的成功日志；检查器重新解析原始回执后，却得到地址为 `Cat`、topic 为 `Purr()` 的空数据日志，于是判定通过。

## 方法总结

这是一道“验证器即攻击面”的题。链上共识认可的语义与题目检查器二次解析得到的语义不一致，攻击者不需要改变 EVM 的事件规则，只需让两套解析器对同一字节串产生不同结构。

审计这类系统时要避免自行拆解共识层编码，尤其不能随意剥离类型字节后交给通用 RLP 解码器。更稳妥的做法是使用与节点相同的 typed-receipt 解码实现，并对完整输入、长度和尾随数据做严格验证。
