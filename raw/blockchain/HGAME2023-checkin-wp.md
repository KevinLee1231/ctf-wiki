# checkin

## 题目简述

题目给出一个部署在测试链上的 `Checkin` 合约。合约内部保存字符串 `greeting`，任何人都可以调用 `setGreeting` 修改它；当 `greeting` 等于 `HelloHGAME!` 时，`isSolved()` 返回 `true`。

核心合约逻辑可以整理为：

```solidity
contract Checkin {
    string greeting;

    constructor(string memory _greeting) {
        greeting = _greeting;
    }

    function setGreeting(string memory _greeting) public {
        greeting = _greeting;
    }

    function isSolved() public view returns (bool) {
        return keccak256(abi.encodePacked("HelloHGAME!"))
            == keccak256(abi.encodePacked(greeting));
    }
}
```

因此，本题不需要寻找重入、整数溢出等漏洞，只需构造一笔交易，把链上状态改成合约期望的字符串。

## 解题过程

从题目实例页面取得 RPC 地址、私钥和合约地址。私钥属于临时比赛环境，不应写进公开题解；下面从环境变量读取这些实例参数。

先安装并导入 `web3.py`，构造只包含 `setGreeting` 与 `isSolved` 的最小 ABI：

```python
import os

from web3 import Web3

RPC_URL = os.environ["HGAME_RPC_URL"]
PRIVATE_KEY = os.environ["HGAME_PRIVATE_KEY"]
CONTRACT_ADDRESS = Web3.to_checksum_address(
    os.environ["HGAME_CONTRACT_ADDRESS"]
)

ABI = [
    {
        "inputs": [{"internalType": "string", "name": "_greeting", "type": "string"}],
        "name": "setGreeting",
        "outputs": [],
        "stateMutability": "nonpayable",
        "type": "function",
    },
    {
        "inputs": [],
        "name": "isSolved",
        "outputs": [{"internalType": "bool", "name": "", "type": "bool"}],
        "stateMutability": "view",
        "type": "function",
    },
]

w3 = Web3(Web3.HTTPProvider(RPC_URL))
account = w3.eth.account.from_key(PRIVATE_KEY)
contract = w3.eth.contract(address=CONTRACT_ADDRESS, abi=ABI)

transaction = contract.functions.setGreeting("HelloHGAME!").build_transaction(
    {
        "from": account.address,
        "nonce": w3.eth.get_transaction_count(account.address),
        "chainId": w3.eth.chain_id,
        "gas": 100_000,
        "gasPrice": w3.eth.gas_price,
    }
)

signed = account.sign_transaction(transaction)
tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
receipt = w3.eth.wait_for_transaction_receipt(tx_hash)

assert receipt.status == 1
assert contract.functions.isSolved().call() is True
print(tx_hash.hex())
```

交易确认后，`greeting` 已被改为 `HelloHGAME!`，此时 `isSolved()` 返回 `true`。回到题目实例页面触发检查即可取得 flag。

## 方法总结

本题考查最基础的智能合约交互流程：阅读状态判定条件、构造函数调用、签名并广播交易，再通过只读调用验证最终状态。分析合约题时应先寻找 `isSolved` 一类判定函数，并反向确定满足条件所需的最小状态变更；不要看到链上环境就预设一定存在复杂漏洞。
