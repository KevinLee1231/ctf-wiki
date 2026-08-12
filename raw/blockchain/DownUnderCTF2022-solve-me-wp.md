# DownUnderCTF 2022 Solve Me Writeup

## 题目简述

题目部署了一个最小化的 Solidity 合约。合约只有公开状态变量 `isSolved` 和一个无参数外部函数 `solveChallenge()`；调用该函数后，`isSolved` 会从 `false` 变为 `true`。平台据此判断实例是否完成。

```solidity
contract SolveMe {
    bool public isSolved = false;

    function solveChallenge() external {
        isSolved = true;
    }
}
```

## 解题过程

从实例接口取得玩家私钥、RPC 地址和合约地址后，用玩家钱包连接私链。调用函数不需要参数或代币，只需发送一笔状态变更交易并等待回执：

```javascript
const provider = ethers.getDefaultProvider(RPC_URL);
const wallet = new ethers.Wallet(playerPrivateKey, provider);
const contract = new ethers.Contract(
    contractAddress,
    ["function solveChallenge()"],
    wallet
);

const tx = await contract.solveChallenge();
await tx.wait();
```

交易确认后，公开 getter `isSolved()` 返回 `true`，平台的 `/challenge/solve` 接口给出：

```text
DUCTF{muM_1_did_a_blonkchain!}
```

## 方法总结

这是一道 EVM 交互入门题，重点不是漏洞利用，而是完成“读取实例信息—构造钱包—按 ABI 调用合约—等待交易确认”的基本流程。遇到链上题时应先找平台实际检查的状态变量，再判断是否存在可以直接把它改到目标值的公开函数。
