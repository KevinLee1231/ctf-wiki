# Play for Free

## 题目简述

题目在 Solana 上部署由 Solang 编译的 Solidity 合约 `Arcade`。合约状态里藏有五枚代币的位置；每次向正确的重载 `find` 函数提交位置，就会在 `tokens` 中设置一位。只有：

```text
tokens == 0x1f
```

时调用 `play()` 才会增加 `playCount`。服务端最终直接读取状态账户偏移 `20..24` 的 `playCount`，大于 0 即返回 flag。

五个位置被声明为 `private`：

```solidity
uint64 private forgotten;
uint64[] private stuckInGap;
uint64[1] private atBottom;
address private somewhere;
string private lookForIt;
```

但链上 `private` 只限制其他 Solidity 合约通过语言级 getter 访问，不会加密 Solana 账户数据。核心是理解 Solang 的固定区与堆区布局，并在自定义求解程序中直接解析目标状态账户。

## 解题过程

### 固定账户顺序

服务允许上传自己的 Solana 程序并指定 account metas。Solang 对构造函数账户顺序有固定假设，因此求解脚本按下列顺序提交：

```python
account_metas = [
    ("user data", "sw"),
    ("user", "sw"),
    ("data account", "-w"),
    ("program", "-r"),
    ("system program", "-r"),
]
```

这样在 `Solve` 构造函数中：

```solidity
AccountInfo memory dataAccount = tx.accounts[2];
```

正好取得 Arcade 的状态账户。它由 Arcade program 拥有，后续 CPI 调用也能满足目标函数的检查：

```solidity
require(
    tx.accounts[0].owner == type(Arcade).program_id,
    "Invalid"
);
```

### 解析 Solang 状态布局

状态账户前 16 字节包含合约选择信息和堆起始偏移。先复制并按以下类型解码：

```solidity
bytes memory metadata = new bytes(16);
for (uint i; i < 16; ++i) {
    metadata[i] = dataAccount.data[i];
}

(, , uint32 heapOffset) =
    abi.decode(metadata, (bytes4, bytes8, uint32));
```

偏移 16 到 `heapOffset` 是固定字段区。动态数组和字符串在这里保存堆指针：

```solidity
bytes memory fixedData = new bytes(heapOffset - 16);
for (uint i = 16; i < heapOffset; ++i) {
    fixedData[i - 16] = dataAccount.data[i];
}

(
    int32 tokens,
    uint32 playCount,
    uint64 forgotten,
    uint32 stuckInGapOffset,
    uint64 atBottom,
    address somewhere,
    uint32 lookForItOffset
) = abi.decode(
    fixedData,
    (int32, uint32, uint64, uint32, uint64, address, uint32)
);
```

`forgotten`、定长数组 `atBottom[0]` 和地址 `somewhere` 已直接恢复。

动态数组 `stuckInGap` 的首元素是堆偏移处的 8 字节：

```solidity
bytes memory gapData = new bytes(8);
for (uint i; i < 8; ++i) {
    gapData[i] = dataAccount.data[stuckInGapOffset + i];
}
uint64 stuckInGap = abi.decode(gapData, (uint64));
```

题目字符串固定为 8 个字符。Solang 在字符串数据前保存长度；官方求解器取 `lookForItOffset - 8` 处的低字节作为长度，再复制 8 字节正文，构造可供 `abi.decode` 使用的缓冲区：

```solidity
bytes memory stringData = new bytes(12);
stringData[0] = dataAccount.data[lookForItOffset - 8];

for (uint i; i < 8; ++i) {
    stringData[i + 4] = dataAccount.data[lookForItOffset + i];
}

string memory lookForIt = abi.decode(stringData, (string));
```

### 收集五枚代币

构造 CPI 时把 Arcade 状态账户作为唯一可写账户传入：

```solidity
AccountMeta[1] metas = [
    AccountMeta({
        pubkey: dataAccount.key,
        is_writable: true,
        is_signer: false
    })
];
```

依次调用三个 `find(string,uint64)`、一个 `find(address)` 和一个 `find(string)`：

```solidity
IArcade arcade = IArcade(arcadeProgram);

arcade.find_string_uint64{accounts: metas}(
    "Token Dispenser",
    forgotten
);
arcade.find_string_uint64{accounts: metas}(
    "Token Counter",
    stuckInGap
);
arcade.find_string_uint64{accounts: metas}(
    "Arcade Machine",
    atBottom
);
arcade.find_bytes32{accounts: metas}(somewhere);
arcade.find_string{accounts: metas}(lookForIt);
arcade.play{accounts: metas}();
```

五次成功分别设置位 `1`、`2`、`4`、`8`、`16`，于是 `tokens` 变成 `0x1f`。`play()` 将 `playCount` 加一，服务端验证通过并返回 flag。官方归档没有给出具体 flag 字符串，因此这里只记录链上可验证的完成状态。

客户端上传 `Solve.so` 后，调用构造函数的数据为：

```python
from base58 import b58decode
from hashlib import sha256

instruction_data = (
    sha256(b"global:new").digest()[:8]
    + b58decode(accounts["program"])
)
```

其中前 8 字节是 Solang 构造函数 discriminator，后面是 Arcade program 地址。

## 方法总结

- 核心技巧：直接解析 Solang 状态账户的固定区、堆指针和动态字段，再通过 CPI 调用目标合约。
- 识别信号：秘密仅由 Solidity `private` 修饰、服务允许上传自定义程序、目标比较 `tx.accounts[0].owner`，说明应利用链上账户可见性和账户列表控制。
- 复用要点：链上 `private` 不是保密机制。分析非原生语言编译到 Solana 的合约时，应先固定账户顺序、头部元数据、字段字节序和动态堆布局，再构造 CPI；服务端最终检查的原始偏移也是很有价值的布局校验点。
