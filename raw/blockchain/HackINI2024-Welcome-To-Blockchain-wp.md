# Welcome To Blockchain

## 题目简述

题目只给出 Ethereum Sepolia 测试网交易哈希 `0xa7d1c8bf96e30371a1049c6332b077eac6877b8b0c6191e362cc1702c5de4c7a`。目标是查询公开交易，并从交易 input calldata 中解码出明文 flag。这是一道链上数据读取入门题，不需要发送交易。

## 解题过程

可以在[该交易的 Sepolia Etherscan 页面](https://sepolia.etherscan.io/tx/0xa7d1c8bf96e30371a1049c6332b077eac6877b8b0c6191e362cc1702c5de4c7a)查看 input，也可以通过 RPC 保留可复现流程：

```bash
TX=0xa7d1c8bf96e30371a1049c6332b077eac6877b8b0c6191e362cc1702c5de4c7a
cast tx "$TX" --rpc-url "$SEPOLIA_RPC"
```

取出返回结果中的 `input` 十六进制字段，去掉开头的 `0x` 后按字节转为 ASCII。若使用 Foundry，可直接执行：

```bash
cast --to-ascii "$INPUT_HEX"
```

解码结果中包含：

```text
shellmates{W3lcOME_tO_3tH3r3um_h4ve_fun}
```

区块浏览器链接只是公开证据入口；关键网络、交易哈希、字段位置和解码方式均已在正文给出，因此不依赖页面仍可完成查询。

## 方法总结

- 核心技巧：通过只读 RPC 或区块浏览器查询公开交易 calldata，并把十六进制字节解码为文本。
- 识别信号：题面只给链名、测试网和交易哈希时，优先检查 input、event logs、internal calls 与状态变化，而不是猜合约漏洞。
- 复用要点：必须使用正确 chain ID 或 RPC；交易哈希在不同网络上没有可互换性，解码前也要区分 ABI 编码数据与直接嵌入的 ASCII。
