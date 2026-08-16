# Contact Me

## 题目简述

题目给出一个最小化的 Solidity `Setup` 合约。状态变量 `solved` 初始为 `false`，外部函数 `makeACall(uint8 _hour)` 只在参数等于 `7` 时把它改为 `true`，`isSolved()` 直接返回该状态。这是一道链上交互入门题，目标不是寻找漏洞，而是正确构造并发送状态变更交易。

## 解题过程

先用只读调用确认初始状态：

```bash
cast call "$SETUP" "solved()(bool)" --rpc-url "$RPC"
```

随后用题目提供的玩家私钥发送交易，调用 `makeACall(uint8)` 并传入 `7`：

```bash
cast send "$SETUP" "makeACall(uint8)" 7 \
  --rpc-url "$RPC" \
  --private-key "$PLAYER_PRIVATE_KEY"
```

交易被打包后再次读取 `isSolved()`：

```bash
cast call "$SETUP" "isSolved()(bool)" --rpc-url "$RPC"
```

返回 `true` 即说明实例已满足取 flag 条件。官方解法给出的验证 flag 为：

```text
shellmates{U_c0ntRACteD_m3_$ucce$sFullY_0x007}
```

## 方法总结

- 核心技巧：使用 ABI 函数签名区分只读 `eth_call` 与会改变状态的签名交易。
- 识别信号：合约公开了明确的状态布尔值和无权限限制的状态修改函数时，先检查是否只是基础交互题。
- 复用要点：`cast call` 不会持久化状态；需要改变链上存储时必须使用 `cast send`、正确私钥并等待交易上链。
