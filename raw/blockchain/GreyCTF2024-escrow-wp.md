# Escrow

## 题目简述

工厂使用 Clones With Immutable Args 部署双资产托管合约，并以

$$
\operatorname{keccak256}(\text{IDENTIFIER},\text{factory},\text{tokenX},\text{tokenY})
$$

作为 escrow NFT 的 token ID。目标托管中有 10000 GREY，但其所有权 NFT 已销毁。需要构造一个参数哈希不同、解码后资产地址却相同的新 clone，使它铸出同一个 token ID，从而重新取得目标托管的所有权。

## 解题过程

CWIA 代理在转发调用时，会把 immutable args 和两字节参数长度追加到原始 calldata 后。正常参数为 factory 20 字节、tokenX 20 字节、tokenY 20 字节，共 60 字节；加上 4 字节 selector 与 2 字节长度，`initialize()` 看到的长度正好是 66。

工厂只按调用者提供的 `implId || args` 计算 `paramsHash`，而 clone 内的 `_getArgAddress(40)` 固定读取 tokenY 的 20 字节。攻击时把 tokenY 从完整的 20 个零字节缩短为 19 个零字节：

```solidity
bytes19 shortZero = bytes19(abi.encodePacked(address(0)));
(uint256 id, ) = factory.deployEscrow(
    0,
    abi.encodePacked(address(grey), shortZero)
);
```

此时 immutable args 总长为 59 字节。tokenY 的第 20 个字节会与尾部两字节长度字段的首字节重叠；59 的两字节表示以 `0x00` 开头，所以读取结果仍是零地址。于是：

- 原参数是 40 字节，攻击参数是 39 字节，`paramsHash` 不同，可绕过 `AlreadyDeployed`；
- clone 解码出的 factory、tokenX、tokenY 与目标完全相同，`escrowId` 相同；
- 原 NFT 已烧毁，新部署过程可把同一 token ID 铸给攻击合约。

目标 escrow 的 `owner()` 查询的是工厂中该 token ID 的当前持有人，所以攻击合约可直接对旧 escrow 调用：

```solidity
DualAssetEscrow(setup.escrow()).withdraw(true, 10_000e18);
```

转出全部 GREY 后得到：

```text
grey{cwia_bytes_overlap_5a392abcfa2d040a}
```

## 方法总结

漏洞来自“唯一性哈希使用原始字节串，而业务 ID 使用解码后的定长字段”。CWIA 又把长度元数据紧邻参数追加，使短一字节的零地址可借用长度字段补齐。审计带尾随 immutable args 的 clone 时，必须同时检查原始 calldata 长度、字段读取边界、尾部元数据和工厂的去重依据，不能只看 ABI 解码后的值。
