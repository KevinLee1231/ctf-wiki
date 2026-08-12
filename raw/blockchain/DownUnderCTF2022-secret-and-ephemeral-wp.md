# DownUnderCTF 2022 Secret and Ephemeral Writeup

## 题目简述

合约构造时接收字符串 `_not_yours` 和整数 `_secret_number`，并把它们与部署者地址一起哈希：

```solidity
constructor(string memory _not_yours, uint256 _secret_number) {
    not_yours = _not_yours;
    spooky_hash = keccak256(
        abi.encodePacked(not_yours, _secret_number, msg.sender)
    );
}

function retrieveTheFunds(
    string memory secret,
    uint256 secret_number,
    address owner_address
) public {
    require(
        keccak256(abi.encodePacked(secret, secret_number, owner_address))
            == spooky_hash
    );
    payable(msg.sender).transfer(address(this).balance);
}
```

变量虽然标记为 `private`，但链上 storage 和历史交易并不保密。目标是恢复构造参数和部署者地址，重放正确的哈希输入。

## 解题过程

`not_yours` 位于 storage slot 3。若字符串超过 31 字节，slot 3 保存 $2L+1$，其中 $L$ 是字节长度；正文数据从 `keccak256(uint256(3))` 开始连续存放。读取相应 storage slot、拼接并去除尾部零字节即可恢复字符串。更直接的办法是读取合约创建交易，因为构造参数以 ABI 编码明文附在 init bytecode 末尾。

从最新区块向前查找 `to == null` 且 `creates == contractAddress` 的创建交易，记录：

- `tx.from`：原始部署者地址；
- `tx.data` 末尾的 ABI 参数：`string` 与 `uint256`。

```javascript
const encoded = "0x" + deploymentTx.data.slice(-320);
const [secret, secretNumber] = ethers.utils.defaultAbiCoder.decode(
    ["string", "uint256"],
    encoded
);

await contract.retrieveTheFunds(
    secret,
    secretNumber,
    deploymentTx.from
);
```

三项输入重新计算出的 `keccak256` 与 `spooky_hash` 相等，合约余额被转给玩家，平台返回：

```text
DUCTF{u_r_a_web3_t1me_7raveler_:)}
```

## 方法总结

Solidity 的 `private` 只限制其它合约通过自动 getter 访问，不能提供机密性。遇到链上“秘密”时，应同时检查固定 storage 布局、动态数组或字符串的数据位置，以及部署和函数调用的历史 calldata。构造参数、部署者地址和状态值都属于公开链上数据，真正的秘密不应直接写入公开链。
