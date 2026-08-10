# Transfer 2

## 题目简述

部署 `Transfer2` 时，构造函数会用固定 salt 通过 `CREATE2` 创建 `Challenge`。只有 `Challenge` 在构造时发现自身余额不少于 `0.5 ether`，才会把 `flag` 设为真并让外层合约触发 `SendFlag` 事件。由于构造函数只执行一次，必须在合约真正部署前预测两个地址，并提前向尚无代码的 `Challenge` 地址转账。

## 解题过程

关键合约为：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.7;

contract Transfer2 {
    Challenge public chall;
    event SendFlag();

    bytes32 constant salt = keccak256("HGAME 2023");

    constructor() {
        chall = new Challenge{salt: salt}();
        if (chall.flag()) {
            emit SendFlag();
        }
    }

    function getCode() public pure returns (bytes memory) {
        return type(Challenge).creationCode;
    }
}

contract Challenge {
    bool public flag;

    constructor() {
        if (address(this).balance >= 0.5 ether) {
            flag = true;
        }
    }
}
```

### 预测外层 `CREATE` 地址

后端以普通 `CREATE` 部署 `Transfer2`。其地址为：

$$
A_T=\operatorname{last}_{20}\left(
\operatorname{keccak256}(\operatorname{RLP}([A_D,\mathit{nonce}]))
\right),
$$

其中 $A_D$ 是部署账户，`nonce` 必须取“部署交易执行时”的值。账户当前没有发过交易时该值通常为 `0`。

### 预测内层 `CREATE2` 地址

内层地址不依赖外层合约的 nonce，而由 EIP-1014 公式确定：

$$
A_C=\operatorname{last}_{20}\left(
\operatorname{keccak256}(0xff\parallel A_T\parallel S\parallel
\operatorname{keccak256}(I))
\right),
$$

其中：

- $S=\operatorname{keccak256}(\texttt{"HGAME 2023"})$；
- $I$ 是 `Challenge` 的 creation code，不是部署后的 runtime bytecode；
- `getCode()`、同版本 Solidity 编译产物或官方给出的字节串都可取得 $I$。

下面的脚本完成两层预测。`Challenge` creation code 必须与远端编译器版本和元数据完全一致：

```python
import rlp
from web3 import Web3

DEPLOYER = "0xfd6Bb138dDd1b2d5aC4d6045A764182b3ED3b245"
DEPLOY_NONCE = 0

CHALLENGE_INIT_CODE = bytes.fromhex(
    "6080604052348015600f57600080fd5b506706f05b59d3b200004710602c"
    "576000805460ff191660011790555b60838061003a6000396000f3fe608060"
    "4052348015600f57600080fd5b506004361060285760003560e01c8063890e"
    "ba6814602d575b600080fd5b60005460399060ff1681565b6040519015158152"
    "60200160405180910390f3fea2646970667358221220c0afce3a78fcc60fe5cb"
    "042db9c8cae10e646b3fcd2f905fa125145eebdf049864736f6c6343000811"
    "0033"
)


def rlp_nonce(nonce):
    if nonce == 0:
        return b""
    return nonce.to_bytes((nonce.bit_length() + 7) // 8, "big")


deployer_bytes = bytes.fromhex(DEPLOYER[2:])
transfer_bytes = Web3.keccak(
    rlp.encode([deployer_bytes, rlp_nonce(DEPLOY_NONCE)])
)[-20:]

salt = Web3.keccak(text="HGAME 2023")
init_code_hash = Web3.keccak(CHALLENGE_INIT_CODE)
challenge_bytes = Web3.keccak(
    b"\xff" + transfer_bytes + salt + init_code_hash
)[-20:]

transfer_address = Web3.to_checksum_address(transfer_bytes.hex())
challenge_address = Web3.to_checksum_address(challenge_bytes.hex())

print("Transfer2:", transfer_address)
print("Challenge:", challenge_address)
```

以这组示例账户和 nonce 计算得到：

```text
Transfer2: 0x021fd257CdCE9A7B4A98e21B56eEa7eE8CF4425b
Challenge: 0x8A65Af3404B37704dc25883B061e65547d934c81
```

先从另一个已入金账户向预测的 `Challenge` 地址转入至少 `0.5 ether`，再让目标部署账户创建 `Transfer2`。只有余额、没有代码和 nonce 的地址仍可被 `CREATE2` 正常部署；构造函数执行时会看到预存余额并设置 `flag = true`。

如果必须由同一个部署账户发送预存款交易，该交易本身会消耗一个 nonce，此时应把 `DEPLOY_NONCE` 设为“当前 nonce + 1”后重新计算两个地址。忽略这一步会把资金转到错误地址。

部署成功后提交对应交易哈希，得到：

```text
hgame{e0638df02eec0ccaa653b66de526c282a335ed3e}
```

最终 flag 与示例地址由 [HGAME2023 官方题解仓库](https://github.com/vidar-team/HGAME2023_Writeup) 收录的参赛者 Week4 题解复核。

## 方法总结

以太坊地址在部署前就可确定，因此“先给未来合约转账”是合法操作。解题时要分清外层 `CREATE` 的 `RLP(sender, nonce)` 与内层 `CREATE2` 的 `0xff || sender || salt || keccak(init_code)`，尤其不能把 runtime bytecode 当成 creation code。任何预转账都会影响发送方 nonce，预测时必须明确每笔交易由哪个账户发出及其顺序。
