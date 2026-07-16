# 肘，上链！

## 题目简述

服务会创建测试账户、部署题目合约，并在 `isSolved()` 返回 `true` 后发放 flag。合约没有权限限制：任何账户都可以调用 `sign(bytes32)` 覆盖状态变量 `signin`。只需提交字符串 `Hello0xBlockchain` 的 Ethereum Keccak-256 值即可满足判断。

## 解题过程

题目合约的关键代码如下：

```solidity
pragma solidity ^0.8.0;

contract Signin {
    bytes32 signin;

    function sign(bytes32 _signin) public {
        signin = _signin;
    }

    function isSolved() public view returns (bool) {
        string memory expected = "Hello0xBlockchain";
        return keccak256(abi.encodePacked(expected)) == signin;
    }
}
```

先连接题目给出的交互服务，选择菜单 `1` 创建部署账户，保存返回的 token 和账户地址，并通过水龙头向该地址转入超过 `0.001` 个测试币。

再次连接服务，选择菜单 `2`，提交刚才的 token，让平台部署挑战合约并返回合约地址。随后把题目 RPC 添加到 MetaMask；RPC URL、链 ID 和账户均使用当前实例给出的值，不应复用旧截图中的临时信息。

使用 Ethereum Keccak-256 计算目标字符串的摘要。注意这里不是 Python `hashlib.sha3_256` 所实现的 NIST SHA3-256。使用 Web3.py 可写为：

```python
from web3 import Web3

target = Web3.keccak(text="Hello0xBlockchain")
print(target.hex())
```

结果为：

```text
83c9a53a09792c2f7d6d0b19bede7af634e365c92cb3874761e2f0ac2f31bd6a
```

在 Remix 中用 Solidity `0.8.x` 编译上述合约接口。

部署环境选择 `Injected Provider - MetaMask`，确认钱包已连接到题目 RPC，然后在 `At Address` 中填入平台返回的合约地址，以已有地址加载合约；此处不要重新部署另一份合约。

调用 `sign`，参数填写带 `0x` 前缀的 32 字节摘要：

```text
0x83c9a53a09792c2f7d6d0b19bede7af634e365c92cb3874761e2f0ac2f31bd6a
```

确认交易并等待上链。

此时调用 `isSolved()` 会返回 `true`。

最后回到交互服务，选择菜单 `3` 并提交当前实例 token，得到：

```text
0xGame{T3st1ng_ur_bl0ckcha1n!}
```

## 方法总结

本题的核心是准确复现 `keccak256(abi.encodePacked("Hello0xBlockchain"))`，再调用无权限限制的 setter 写入该值。平台账户、RPC、token 和合约地址都属于临时实例信息；WP 中只保留稳定的合约机制、摘要和交互顺序。
