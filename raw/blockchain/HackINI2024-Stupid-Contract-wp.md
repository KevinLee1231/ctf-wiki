# Stupid Contract

## 题目简述

`Setup` 不部署可读 Solidity 目标，而是用 `create` 部署一段手写 EVM creation bytecode。`Setup.isSolved()` 随后按 `isSolved()` ABI 调用目标并期待返回 `true`。需要把 creation code 与 runtime code 分开，阅读 opcode，找出能把 storage slot 0 设为 `1` 的 selector 和附加调用条件。

## 解题过程

给出的开头为：

```text
60 60 80 60 0b 60 00 39 60 00 f3
```

它把从偏移 `0x0b` 开始、长度 `0x60` 的数据复制到内存并 `RETURN`，因此后续字节才是最终 runtime。对 runtime 反汇编后可归纳出两条分支：

```text
selector 0xc2865037:
    要求 msg.value != 0
    SSTORE(0, 1)

selector 0x64d98f6e:
    SLOAD(0)
    返回 32 字节结果
```

`0xc2865037` 对应 `dumpBump()`；另一 selector 是 `isSolved()`。先取目标地址，再向 `dumpBump()` 发送任意非零 value。官方示例使用 `1 ether`：

```bash
TARGET=$(cast call "$SETUP" "TARGET()(address)" --rpc-url "$RPC")

cast send "$TARGET" "dumpBump()" \
  --value 1ether \
  --rpc-url "$RPC" \
  --private-key "$PLAYER_PRIVATE_KEY"
```

这会执行 `SSTORE`，把 slot 0 写成 `1`。再调用 Setup 的检查函数：

```bash
cast call "$SETUP" "isSolved()(bool)" --rpc-url "$RPC"
```

返回 `true` 后可取得：

```text
shellmates{y0u_S33_evm_0PC0d3s_4r3_L1KE_4S$EmBly}
```

## 方法总结

- 核心技巧：识别 creation bytecode 的 `CODECOPY + RETURN`，只对实际 runtime 做控制流和 storage 语义还原。
- 识别信号：合约通过 `create` 直接部署 hex bytecode、ABI 缺失或 selector 只有四字节常量时，应从 dispatcher、`SLOAD`、`SSTORE`、`CALLVALUE` 等关键 opcode 建立伪代码。
- 复用要点：不要简单按第一个 `f3` 猜边界；应结合 `CODECOPY` 的 offset 和 length 验证 runtime 范围，并分别记录 selector、value 条件和状态效果。
