# babyblock

## 题目简述


原始方向为 `Blockchain`；归档时按主要障碍归入 `web`，因为解法主体是 Solidity/Web3 智能合约状态逻辑审计与链上调用。

题目给出一个简单 Solidity 合约 `Challenge`。构造函数把 `secretNumber` 设置为 `block.timestamp % 10 + 1`，看起来需要猜中 1 到 10 的随机数；但 `guessNumber()` 的 assembly 实际只比较 `secretNumber` 与输入 `_num` 的最低位是否相同，并把比较结果写入 `solved`。

关键合约逻辑如下：

```solidity
constructor() public {
    secretNumber = block.timestamp % 10 + 1;
}

function guessNumber(uint256 _num) public {
    uint256 num = _num;

    assembly {
        let m := mload(0x40)
        let a := and(sload(secretNumber_slot), 1)
        let b := and(num, 1)
        let result := eq(a, b)
        mstore(m, result)
        sstore(solved_slot, result)
    }
}
```

因此本题不是预测完整随机数，而是让输入数和 `secretNumber` 的奇偶性一致。

## 解题过程

```python
from web3 import Web3, HTTPProvider
from web3.middleware import geth_poa_middleware

w3 = Web3(HTTPProvider('http://localhost:8545'))
w3.middleware_onion.inject(geth_poa_middleware, layer=0)

abi = """
[
	{
		"inputs": [],
		"payable": false,
		"stateMutability": "nonpayable",
		"type": "constructor"
	},
	{
		"constant": false,
		"inputs": [
			{
				"internalType": "uint256",
				"name": "_num",
				"type": "uint256"
			}
		],
		"name": "guessNumber",
		"outputs": [],
		"payable": false,
		"stateMutability": "nonpayable",
		"type": "function"
	},
	{
		"constant": true,
		"inputs": [],
		"name": "isSolved",
		"outputs": [
			{
				"internalType": "bool",
				"name": "",
				"type": "bool"
			}
		],
		"payable": false,
		"stateMutability": "view",
		"type": "function"
	},
	{
		"constant": true,
		"inputs": [],
		"name": "solved",
		"outputs": [
			{
				"internalType": "bool",
				"name": "",
				"type": "bool"
			}
		],
		"payable": false,
		"stateMutability": "view",
		"type": "function"
	}
]
"""

privatekey = 0x25ece0fd60eb70e6216623edf3a6f11f38f690cac80a4d7a35f4c60648e25043
acct = w3.eth.account.from_key(privatekey)
address = w3.eth.account.from_key(privatekey).address
# print(address)

#获取余额
# address = "0x1F933E837C02eE03497129e7C378b0BB9D502809"
contractaddr = "0x44e406030B4A55DF1db1De7dE01dfe3b1a05d908"

contract = w3.eth.contract(address=contractaddr, abi=abi)

guessed_number = w3.eth.get_block('latest')['timestamp'] % 10 + 1

# 如果block.timestamp的LSB不同于guessedNumber，那么加1
if (w3.eth.get_block('latest')['timestamp'] & 1) != (guessed_number & 1):
    guessed_number += 1

contract_txn = contract.functions.guessNumber(guessed_number).build_transaction({
    'from': address,
    'nonce': w3.eth.get_transaction_count(address),
    'gas': 1000000,
    'gasPrice': w3.to_wei('1', 'gwei')
})

signed_txn = acct.sign_transaction(contract_txn)
txn_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)
txn_receipt = w3.eth.wait_for_transaction_receipt(txn_hash)
print(txn_receipt)

print(contract.functions.isSolved().call())
```

脚本从最新区块时间计算 `block.timestamp % 10 + 1` 作为候选值。由于合约只比较最低位，候选值不需要完全等于 `secretNumber`，只要奇偶性一致即可；若当前区块时间最低位和候选值最低位不一致，就把候选值加 1 修正奇偶性。之后调用 `guessNumber()`，再用 `isSolved()` 验证状态是否为 `true`。

## 方法总结

- 核心技巧：不要被“猜随机数”的表象干扰，实际阅读 assembly，确认合约只校验最低位。
- 识别信号：Solidity 题中出现 `assembly`、`sload(slot)`、`and(x, 1)`、`sstore(solved_slot, result)` 时，应优先判断真正写入 `solved` 的条件。
- 复用要点：区块时间随机数题不一定需要预测精确值；如果检查逻辑弱化为 parity、范围或哈希前缀，只满足被检查部分即可。
