# smart-greetings

## 题目简述

题目提供 Solidity 合约 `Bouncer`。五个 `address` 类型的私有数组元素实际承载 20 字节文本；读取它们需要先通过 `bribe()` 成为 `friends`，再以固定答案调用 `get_flag()`。核心是合约状态与链上调用语义，因此归入 Blockchain。

## 解题过程

合约的两个条件非常直接：

```solidity
function bribe() external payable {
    if (msg.value > 10000) friends[msg.sender] = true;
}

function get_flag(uint index, uint answer) external view returns (address) {
    if (answer == 0x31337 && friends[msg.sender]) return theFlags[index];
    return address(0);
}
```

用同一钱包发送 10001 wei 调用 `bribe()`，等待交易确认，然后以该钱包地址作为调用者遍历索引 0 至 4：

```javascript
let tx = await contract.bribe({value: 10001});
await tx.wait();

for (let i = 0; i < 5; i++) {
  console.log(await contract.get_flag(i, 0x31337));
}
```

迁移脚本表明索引 0、1、4 是 `never gonna gib u up` 的诱饵，索引 2、3 分别返回：

```text
0x67726579686174737b5930755f6172335f615f53
0x6d3472745f503372736f6e7d0000000000000000
```

去掉 `0x` 后按十六进制转字节，删除第二段末尾用于补足 20 字节的零字节并拼接，得到：

```text
greyhats{Y0u_ar3_a_Sm4rt_P3rson}
```

## 方法总结

Solidity 的 `address` 不只可以表示账户，也能被题目当作定长 20 字节容器。调用前必须满足链上状态条件：转入值是严格大于 10000，而不是大于等于；写交易与后续只读调用还应使用同一地址。解码时按原始 20 字节处理，不能把它先转成会丢失前导零的普通整数。
